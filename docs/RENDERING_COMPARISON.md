# Rendering System Comparison: tmux vs zellij

This document provides a detailed technical comparison of the rendering architectures used by tmux and zellij, with a focus on understanding why zellij achieves flicker-free rendering while tmux exhibits flickering when running TUI applications like Claude Code.

## Executive Summary

| Aspect | tmux | zellij |
|--------|------|--------|
| Language | C | Rust |
| Dirty Tracking | None (full redraws) | Per-line HashSet |
| Render Debouncing | None | 10ms coalescing |
| Differential Rendering | No | Yes |
| Synchronized Output | Yes (CSI ?2026) | Yes (CSI ?2026) |
| Cursor Management | During render | Hide during, show after |

The key difference is that **zellij tracks which lines have changed and only redraws those lines**, while **tmux redraws entire panes on every update**. This fundamental architectural difference is the primary cause of flickering in tmux.

---

## tmux Rendering Architecture

### Core Files and Responsibilities

| File | Purpose |
|------|---------|
| `grid.c` | Grid data structure storing cell content |
| `screen.c` | Screen abstraction over grid with cursor state |
| `screen-write.c` | API for writing to screens (used by applications) |
| `screen-redraw.c` | Orchestrates redrawing screens to terminal |
| `tty.c` | Low-level terminal output and escape sequences |
| `server-client.c` | Client management and redraw scheduling |

### Data Flow

```
Application Output
       ↓
   screen-write.c (writes to grid)
       ↓
   server-client.c (schedules redraw)
       ↓
   screen-redraw.c (orchestrates full redraw)
       ↓
   tty.c (outputs escape sequences)
       ↓
   Terminal
```

### Key Data Structures

#### struct grid (grid.c)
```c
struct grid {
    int              flags;
    u_int            sx;        /* grid width */
    u_int            sy;        /* grid height */
    u_int            hscrolled; /* lines scrolled into history */
    u_int            hsize;     /* history size */
    u_int            hlimit;    /* history limit */
    struct grid_line *linedata; /* array of lines */
};
```

#### struct grid_line (tmux.h)
```c
struct grid_line {
    u_int             cellused;  /* cells actually used */
    u_int             cellsize;  /* cells allocated */
    struct grid_cell *celldata;  /* cell array */
    u_int             extdsize;  /* extended data size */
    struct grid_extd *extddata;  /* extended cell data */
    int               flags;     /* line flags */
};
```

#### struct screen (tmux.h)
```c
struct screen {
    char             *title;
    char             *path;
    u_int             cx;        /* cursor x */
    u_int             cy;        /* cursor y */
    u_int             sx;        /* screen width */
    u_int             sy;        /* screen height */
    struct grid      *grid;      /* underlying grid */
    /* ... other fields ... */
};
```

### Rendering Process (screen-redraw.c)

The main rendering entry point is `screen_redraw_screen()`:

1. **Setup Phase**
   - Check if redraw is needed via client flags
   - Start synchronized output (`tty_sync_start`)
   - Hide cursor (`TTY_NOCURSOR`)

2. **Pane Rendering** (`screen_redraw_draw_pane`)
   - For each line in the pane (0 to `wp->sy`):
     - Position cursor at line start
     - Call `tty_draw_line()` to output the entire line
   - **No dirty checking** - every line is redrawn

3. **Border/Status Rendering**
   - Draw pane borders
   - Draw status line
   - Draw message/prompt if active

4. **Cleanup Phase**
   - Restore cursor position
   - End synchronized output (`tty_sync_end`)
   - Show cursor

### Line Drawing (tty.c)

`tty_draw_line()` outputs a single line:

```c
void tty_draw_line(struct tty *tty, struct screen *s, u_int px, u_int py,
                   u_int nx, u_int aty, /* ... */)
{
    /* Position cursor */
    tty_cursor(tty, px, aty);

    /* For each cell in the line */
    for (i = 0; i < nx; i++) {
        gc = grid_view_peek_cell(gd, px + i, py);
        /* Output cell with appropriate attributes */
        tty_cell(tty, gc, ...);
    }

    /* Clear to end of line if needed */
    if (clear)
        tty_term_putcode(tty, TTYC_EL);
}
```

### Synchronized Output

tmux supports synchronized output via terminfo capability `Sync`:

```c
void tty_sync_start(struct tty *tty) {
    if (tty_term_has(tty->term, TTYC_SYNC))
        tty_putcode_i(tty, TTYC_SYNC, 1);  /* CSI ?2026h */
}

void tty_sync_end(struct tty *tty) {
    if (tty_term_has(tty->term, TTYC_SYNC))
        tty_putcode_i(tty, TTYC_SYNC, 2);  /* CSI ?2026l */
}
```

### Problems Identified

1. **No Dirty Tracking**: Every redraw outputs every line, even unchanged ones
2. **Multiple Separate tty_write Calls**: Each cell/attribute can trigger a write
3. **Full Pane Redraws**: `PANE_REDRAW` flag triggers complete pane redraw
4. **No Debouncing**: Rapid updates trigger rapid redraws
5. **Sync Nesting Issues**: Nested sync start/end calls can break synchronization

---

## zellij Rendering Architecture

### Core Files and Responsibilities

| File | Purpose |
|------|---------|
| `zellij-server/src/panes/grid.rs` | Grid data with dirty tracking |
| `zellij-server/src/panes/terminal_pane.rs` | Pane rendering logic |
| `zellij-server/src/output/mod.rs` | Output buffering and rendering |
| `zellij-server/src/screen.rs` | Screen management and debouncing |

### Data Flow

```
Application Output
       ↓
   grid.rs (writes to grid, marks dirty)
       ↓
   screen.rs (debounces render requests)
       ↓
   [10ms delay]
       ↓
   terminal_pane.rs (renders only dirty lines)
       ↓
   output/mod.rs (buffers and outputs)
       ↓
   Terminal
```

### Key Data Structures

#### Grid (grid.rs)
```rust
pub struct Grid {
    lines_above: VecDeque<Row>,    // scrollback
    viewport: Vec<Row>,             // visible area
    cursor: Cursor,
    width: usize,
    height: usize,
    pending_messages_to_pty: Vec<Vec<u8>>,
    // ... other fields
}
```

#### Row with Change Tracking (grid.rs)
```rust
pub struct Row {
    columns: Vec<TerminalCharacter>,
    is_canonical: bool,
}
```

#### Dirty Line Tracking (terminal_pane.rs)
```rust
pub struct TerminalPane {
    grid: Grid,
    // Lines that need redrawing
    changed_since_last_render: HashSet<usize>,
    // ... other fields
}
```

### Dirty Tracking Implementation

When content changes, zellij marks specific lines as dirty:

```rust
// In grid.rs - when adding/modifying content
fn mark_line_dirty(&mut self, line_index: usize) {
    self.changed_since_last_render.insert(line_index);
}

// Called from various modification methods:
// - add_character()
// - move_cursor()
// - clear_line()
// - scroll_up/down()
```

### Render Debouncing (screen.rs)

```rust
const REPAINT_DELAY_MS: u64 = 10;

fn schedule_render(&mut self) {
    if !self.render_scheduled {
        self.render_scheduled = true;
        // Schedule render after 10ms delay
        schedule_timer(REPAINT_DELAY_MS, || {
            self.perform_render();
        });
    }
}
```

This coalesces rapid updates into single render operations.

### Differential Rendering (terminal_pane.rs)

```rust
fn render(&mut self, output: &mut Output) -> Result<()> {
    // Start synchronized output
    output.push_str("\x1b[?2026h");

    // Hide cursor during render
    output.push_str("\x1b[?25l");

    // Only render dirty lines
    for line_index in self.changed_since_last_render.drain() {
        self.render_line(line_index, output)?;
    }

    // Show cursor
    output.push_str("\x1b[?25h");

    // End synchronized output
    output.push_str("\x1b[?2026l");

    Ok(())
}

fn render_line(&self, y: usize, output: &mut Output) -> Result<()> {
    // Position cursor at line start
    output.push_str(&format!("\x1b[{};1H", y + 1));

    // Output line content
    let row = &self.grid.viewport[y];
    for cell in &row.columns {
        output.push_character(cell);
    }

    // Clear to end of line
    output.push_str("\x1b[K");

    Ok(())
}
```

### Output Buffering (output/mod.rs)

zellij buffers all output before writing:

```rust
pub struct Output {
    buffer: String,
}

impl Output {
    pub fn flush(&mut self, writer: &mut impl Write) -> Result<()> {
        // Single write call for entire frame
        writer.write_all(self.buffer.as_bytes())?;
        self.buffer.clear();
        Ok(())
    }
}
```

### Synchronized Output

zellij always uses CSI ?2026 sequences:

```rust
// Start of render
output.push_str("\x1b[?2026h");  // Begin synchronized update

// ... render content ...

// End of render
output.push_str("\x1b[?2026l");  // End synchronized update
```

---

## Key Differences Analysis

### 1. Dirty Line Tracking

**tmux**: No tracking. Every redraw iterates through every line and outputs it.

**zellij**: Uses `HashSet<usize>` to track which line indices have changed. Only those lines are rendered.

**Impact**: For a 50-line pane where 1 line changes:
- tmux outputs ~50 lines of escape sequences
- zellij outputs ~1 line of escape sequences

### 2. Render Debouncing

**tmux**: Renders immediately on every update. If an application outputs 100 updates in 10ms, tmux renders 100 times.

**zellij**: Waits 10ms after first update, then renders once with all accumulated changes.

**Impact**: Reduces render calls by 10-100x for rapidly updating applications.

### 3. Output Buffering

**tmux**: Multiple `tty_write()` calls during a single render. Each attribute change, each cell, can trigger a write.

**zellij**: Builds entire frame in a string buffer, then flushes with single write.

**Impact**: Fewer syscalls, more atomic updates to terminal.

### 4. Cursor Management

**tmux**: Cursor hidden via `TTY_NOCURSOR` flag during render, but implementation varies.

**zellij**: Explicit `\x1b[?25l` (hide) at render start, `\x1b[?25h` (show) at render end.

**Impact**: Prevents cursor flicker during partial updates.

### 5. Synchronized Output Handling

**tmux**: Uses terminfo `Sync` capability, requires terminal support detection.

**zellij**: Always outputs CSI ?2026h/l sequences directly.

**Impact**: zellij works with any terminal supporting the sequence, tmux may miss terminals without terminfo entry.

---

## Flickering Root Causes

### Why tmux Flickers

1. **Full Redraws**: When a single character changes, tmux redraws the entire pane
2. **Multiple Writes**: Each line is written separately, allowing terminal to display intermediate states
3. **No Coalescing**: Rapid application output triggers rapid full redraws
4. **Sync Gaps**: Between pane redraws, sync may end and restart

### Why zellij Doesn't Flicker

1. **Minimal Output**: Only changed lines are redrawn
2. **Single Write**: Entire frame buffered and written atomically
3. **Debouncing**: Rapid updates coalesced into single render
4. **Continuous Sync**: Sync block wraps entire render operation

---

## Recommendations for tmux

Based on this analysis, the following changes would reduce flickering in tmux:

### Phase 1: Dirty Line Tracking
- Add `dirty_generation` counter to `struct grid_line`
- Add global `generation` counter to `struct grid`
- Mark lines dirty when modified in `grid_set_cell()`, `grid_clear()`, etc.

### Phase 2: Differential Rendering
- Add `rendered_generations` array to `struct screen`
- In `screen_redraw_draw_pane()`, skip lines where `dirty_generation <= rendered_generation`
- Mark lines clean after rendering

### Phase 3: Render Debouncing
- Add render timer to `struct client`
- Instead of immediate redraw, schedule render after 10ms
- Cancel and reschedule on new updates within window

### Phase 4: Output Buffering Enhancement
- Consider buffering entire pane render before writing
- Reduce number of separate `tty_write()` calls

### Phase 5: Sync Improvements
- Add fallback CSI ?2026h/l when terminfo lacks Sync
- Implement reference counting for nested sync blocks
- Ensure single sync block covers all pane redraws

---

## Appendix: Escape Sequence Reference

### Cursor Control
| Sequence | Description |
|----------|-------------|
| `\x1b[?25l` | Hide cursor |
| `\x1b[?25h` | Show cursor |
| `\x1b[{row};{col}H` | Position cursor |

### Line Operations
| Sequence | Description |
|----------|-------------|
| `\x1b[K` | Erase to end of line (EL) |
| `\x1b[2K` | Erase entire line |
| `\x1b[{n}X` | Erase {n} characters (ECH) |

### Synchronized Output (DEC Private Mode 2026)
| Sequence | Description |
|----------|-------------|
| `\x1b[?2026h` | Begin synchronized update |
| `\x1b[?2026l` | End synchronized update |

Synchronized output tells the terminal to buffer all output until the end sequence, then display atomically. This prevents partial frame display.

---

## References

- [tmux source code](https://github.com/tmux/tmux)
- [zellij source code](https://github.com/zellij-org/zellij)
- [XTerm Control Sequences](https://invisible-island.net/xterm/ctlseqs/ctlseqs.html)
- [Terminal Synchronized Output](https://gist.github.com/christianparpart/d8a62cc1ab659194337d73e399004036)
