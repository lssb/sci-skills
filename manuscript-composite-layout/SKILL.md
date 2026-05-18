---
name: manuscript-composite-layout
description: >-
  Assembles journal **whole figures** from `<bundle>/figures/design.txt` using a **single
  layout model**: treat the file as a **character cell matrix** (one char per column per
  line), compute each panel letter’s **axis-aligned bounding box** in **NPC** coordinates,
  then **`cowplot::ggdraw` + `draw_plot`** (+ **`draw_label`** for panel tags). **R helpers (`manuscript_*` layout API)** are **embedded in this skill** (no dependency on another repo file). Covers **# spacer** / tag-only slots, optional **journal tag
  remap** (design letter ≠ printed tag), **ggsave** width/height/scale per project,
  **bbox CSV** QA, optional **pixel checks**, and **Illustrator** hybrid handoff. Run with
  **`conda activate R`**. Pair with **`manuscript-figure-curation`** for per-panel
  `build_*` scripts, yaml, and **`figure_manifest.yaml`**. Use when the user edits
  `design.txt`, adds `Fig0N_composite_drawplot.R`, or compares composite exports.

---
# Manuscript composite layout (`design.txt` → NPC `draw_plot`)

## Relationship to **manuscript-figure-curation**

- **manuscript-figure-curation** — per-panel **`build_<manifest_id>(root)`** scripts, bundle layout, **`figure_manifest.yaml`**, typography on **收录**, **重画刊例** / **fig-spec refresh** for panels.
- **This skill** — **one** assembly model for the **full figure**: cell matrix → bbox → **`draw_plot`**. Do **not** reintroduce legacy **nested `cowplot::plot_grid` by stripe / RLE**, thin-bridge blanking tricks, or GIJK half-height nesting unless you encode that geometry **explicitly** in the **cell grid** (letters only where the plot should appear).

## When this applies

- Editing or validating **`<bundle>/figures/design.txt`**
- Authoring **`Fig0N_composite_drawplot.R`** (or bundle-specific name) next to legacy `Fig0N_composite.R` during migration
- Explaining **NPC** placement, **rectangle asserts**, **`#`** handling, or **tag remaps**
- **`ggsave`** dimensions / scale for a **180 mm**-wide composite (or project default)

## `design.txt` rules

- Every non-empty line has the **same** character count (**`ncol`**).
- Allowed characters: **`#`** (spacer / empty cell) and **`A`–`Z`** (panel slot ids).
- Each **physical line** is one **matrix row**; each **character** is one **matrix column**.
- **Wide or tall panels**: repeat the same letter across columns and/or rows so the letter occupies a **solid axis-aligned rectangle** in the matrix.

## Algorithm (NPC bounding boxes)

1. Read all non-empty lines into a character matrix **`mat`** with **`n_rows`** rows and **`n_cols`** columns.
2. For each letter **`L`** in **`mat`** except **`#`**:
   - `idx <- which(mat == L, arr.ind = TRUE)`
   - `row_min <- min(idx[,"row"])` … `row_max`, `col_min` … `col_max`
   - **Assert** `mat[row_min:row_max, col_min:col_max]` contains **only** `L` (one rectangle per id; otherwise **`stop`**).
   - **NPC** (origin **bottom-left**; matrix row **1 = top** of figure):
     - `width  <- (col_max - col_min + 1) / n_cols`
     - `height <- (row_max - row_min + 1) / n_rows`
     - `x <- (col_min - 1) / n_cols`
     - `y <- 1 - (row_max / n_rows)`
3. `p <- cowplot::ggdraw()`; for each letter (stable order): `p <- p + cowplot::draw_plot(plot_list[[L]], x, y, width, height)`.
4. **Panel tags:** `cowplot::draw_label` at the top-left of each bbox (typical **`label_args`**: Arial bold **11 pt**, `hjust = -0.1`, `vjust = 1.1` to match `.cursorrules` §2C panel tags). If **printed tag ≠ design letter**, remap in the composite script (see **Tag remap** below).

**Figure aspect ratio:** `height / width == n_rows / n_cols` when the grid is full.

## `#` spacers and tag-only cells

- **`#`** cells **do not** receive `draw_plot` from the bbox table (only letters are listed).
- If the legacy layout used **`ggdraw()` + label `"A"`** in a **`#`** column, use an **in-memory-only** copy of the matrix: replace that **`#`** block with **`A`**, set **`plot_list[["A"]] <- cowplot::ggdraw()`**, and let **`draw_label`** print **`A`**.

## Tag remap (design letter → printed tag)

When the journal uses **continuous A…N** on the figure but **`design.txt`** uses **gaps** (e.g. design **`K`** should print **`J`**), build a **named vector** `tag_map` (names = design letters, values = printed tags). In the **`draw_label`** loop, use **`if (letter %in% names(tag_map)) unname(tag_map[[letter]]) else letter`**, or extend **`manuscript_cowplot_assemble_from_design_matrix`** in your bundle with an optional **`tag_map`** argument.

```r
tag_map <- c(K = "J", L = "K")  # example only; names must match design.txt letters
label_for <- function(letter) if (letter %in% names(tag_map)) unname(tag_map[[letter]]) else letter
```

## Embedded R — **`manuscript_*`** (`design.txt` → bbox → `draw_plot` + `draw_label`)

Copy into **`<bundle>/scripts/`** as-is or **`source()`** once per composite script. **Requires** `cowplot`.

```r
# design.txt cell grid -> NPC bounding boxes -> cowplot::ggdraw + draw_plot + draw_label
# Public API:
#   manuscript_read_design_char_matrix(path)
#   manuscript_design_matrix_panel_bbox_table(mat)
#   manuscript_design_matrix_assert_rectangular_panels(mat, letters)
#   manuscript_cowplot_assemble_from_design_matrix(mat, plot_list, label_args)

manuscript_read_design_char_matrix <- function(path) {
    if (!file.exists(path)) stop("Missing design file: ", path)
    lines <- readLines(path, warn = FALSE)
    lines <- lines[nzchar(lines)]
    if (!length(lines)) stop("design.txt is empty: ", path)
    nc <- nchar(lines[[1L]])
    bad <- which(vapply(lines, nchar, 0L) != nc)
    if (length(bad)) {
        stop(
            "design.txt: every line must have length ", nc, "; offending line index(es): ",
            paste(utils::head(bad, 20L), collapse = ", "),
            if (length(bad) > 20L) ", ..." else "",
            ". Fix ", path
        )
    }
    raw <- paste0(lines, collapse = "")
    if (grepl("[^#A-Z]", raw, perl = TRUE)) {
        stop("design.txt: only '#' and uppercase A-Z are allowed. Fix ", path)
    }
    mat <- do.call(rbind, lapply(lines, function(line) strsplit(line, "")[[1]]))
    storage.mode(mat) <- "character"
    mat
}

manuscript_design_matrix_panel_bbox_table <- function(mat) {
    n_rows <- nrow(mat)
    n_cols <- ncol(mat)
    letters <- sort(setdiff(unique(as.vector(mat)), "#"))
    rows <- lapply(letters, function(plot_name) {
        idx <- which(mat == plot_name, arr.ind = TRUE)
        if (!nrow(idx)) {
            return(NULL)
        }
        row_min <- min(idx[, "row"])
        row_max <- max(idx[, "row"])
        col_min <- min(idx[, "col"])
        col_max <- max(idx[, "col"])
        width <- (col_max - col_min + 1L) / n_cols
        height <- (row_max - row_min + 1L) / n_rows
        x <- (col_min - 1L) / n_cols
        y <- 1 - (row_max / n_rows)
        data.frame(
            letter = plot_name,
            row_min = row_min,
            row_max = row_max,
            col_min = col_min,
            col_max = col_max,
            x = x,
            y = y,
            width = width,
            height = height,
            stringsAsFactors = FALSE
        )
    })
    rows <- rows[!vapply(rows, is.null, logical(1))]
    if (!length(rows)) {
        return(data.frame())
    }
    do.call(rbind, rows)
}

manuscript_design_matrix_assert_rectangular_panels <- function(mat, letters) {
    for (plot_name in letters) {
        idx <- which(mat == plot_name, arr.ind = TRUE)
        if (!nrow(idx)) next
        row_min <- min(idx[, "row"])
        row_max <- max(idx[, "row"])
        col_min <- min(idx[, "col"])
        col_max <- max(idx[, "col"])
        sub <- mat[row_min:row_max, col_min:col_max, drop = FALSE]
        bad <- which(sub != plot_name, arr.ind = TRUE)
        if (nrow(bad)) {
            stop(
                "Design letter '", plot_name, "' is not a solid rectangle in the cell grid ",
                "(row ", row_min, ":", row_max, ", col ", col_min, ":", col_max, "). ",
                "Patchwork-style bbox assembly requires one orthotopic rectangle per id."
            )
        }
    }
    invisible(NULL)
}

manuscript_cowplot_assemble_from_design_matrix <- function(mat, plot_list, label_args) {
    n_rows <- nrow(mat)
    n_cols <- ncol(mat)
    letters_in_mat <- setdiff(unique(as.vector(mat)), "#")
    manuscript_design_matrix_assert_rectangular_panels(mat, letters_in_mat)

    tbl <- manuscript_design_matrix_panel_bbox_table(mat)
    if (!nrow(tbl)) {
        stop("manuscript_cowplot_assemble_from_design_matrix: no panel letters in matrix.")
    }

    missing <- setdiff(tbl$letter, names(plot_list))
    if (length(missing)) {
        stop(
            "plot_list is missing named grobs for design letter(s): ",
            paste(missing, collapse = ", ")
        )
    }

    p <- cowplot::ggdraw()

    for (k in seq_len(nrow(tbl))) {
        row <- tbl[k, ]
        letter <- row$letter[[1L]]
        plt <- plot_list[[letter]]
        if (is.null(plt)) {
            stop("plot_list[[\"", letter, "\"]] is NULL.")
        }
        p <- p + cowplot::draw_plot(
            plt,
            x = row$x,
            y = row$y,
            width = row$width,
            height = row$height
        )
    }

    ff <- label_args$label_fontfamily
    fs <- label_args$label_size
    fc <- label_args$label_fontface
    hj <- label_args$hjust
    vj <- label_args$vjust

    for (k in seq_len(nrow(tbl))) {
        row <- tbl[k, ]
        letter <- row$letter[[1L]]
        lx <- row$x + 0.01
        ly <- row$y + row$height
        p <- p + cowplot::draw_label(
            letter,
            x = lx,
            y = ly,
            fontfamily = ff,
            fontface = fc,
            size = fs,
            hjust = hj,
            vjust = vj
        )
    }

    p
}
```

## Deliverables and manifest

- **Exports:** e.g. **`F*/figures/Fig0N_composite_drawplot.png`** and **`.pdf`** (keep legacy stems until the project retires them).
- **Log:** **`F*/scripts/Fig0N_composite_drawplot.R.log`** (same basename rule as other R scripts if the project requires it).
- **Optional QA:** **`F*/scripts/Fig0N_composite_drawplot_layout_bbox.csv`** from **`manuscript_design_matrix_panel_bbox_table(mat)`**.
- Register in **`figure_manifest.yaml`** (see **manuscript-figure-curation** for entry shape).

## Export defaults (align with `.cursorrules`)

- **Width:** **`W_IN <- 180 / 25.4`** inches when the composite is **journal full width**.
- **Height:** **`h_in <- W_IN * n_rows / n_cols`** (same abstract grid as the matrix).
- **`scale`**, **`dpi`**, **`device = cairo_pdf`:** follow the **bundle’s existing composite** policy (many projects use **`scale = 1.3`**; others use a higher scale for dense grids) — do not silently change scale when migrating only the **assembler**.

## Illustrator hybrid

Mechanism / pathway art is often finished in **Adobe Illustrator**. Export the **R** composite at **180 mm** width, then place beside mechanism art at **uniform scale** and matching **pt** sizes (see `.cursorrules` §2C / §3).

## Pixel-level verification (optional)

```python
from PIL import Image
import numpy as np
gray = np.array(Image.open("composite.png").convert("L"))
# Compare landmark x/y (e.g. panel borders) against bbox CSV expectations.
```

## Reference code — inline string layout (`cowplot_design`)

For quick experiments **without** a `design.txt` file on disk: same **NPC** math as **## Algorithm** and **`manuscript_*`**. For production, prefer **`manuscript_read_design_char_matrix`** + **`manuscript_cowplot_assemble_from_design_matrix`** (embedded above).

```r
library(ggplot2)
library(cowplot)

# design_str: layout e.g. "AAB\nCCC"; plot_list: named list(A = ggplot, ...)
cowplot_design <- function(design_str, plot_list) {
  lines <- strsplit(trimws(design_str), "\n")[[1]]
  lines <- lines[lines != ""]

  mat <- do.call(rbind, lapply(lines, function(x) {
    strsplit(gsub(" ", "", x), "")[[1]]
  }))

  n_rows <- nrow(mat)
  n_cols <- ncol(mat)
  unique_plots <- setdiff(unique(as.vector(mat)), "#")
  p <- ggdraw()

  for (plot_name in unique_plots) {
    indices <- which(mat == plot_name, arr.ind = TRUE)
    row_min <- min(indices[, "row"])
    row_max <- max(indices[, "row"])
    col_min <- min(indices[, "col"])
    col_max <- max(indices[, "col"])
    width <- (col_max - col_min + 1) / n_cols
    height <- (row_max - row_min + 1) / n_rows
    x <- (col_min - 1) / n_cols
    y <- 1 - (row_max / n_rows)
    current_plot <- plot_list[[plot_name]]
    if (!is.null(current_plot)) {
      p <- p + draw_plot(current_plot, x = x, y = y, width = width, height = height)
    } else {
      warning("plot_list has no entry for design letter: ", plot_name)
    }
  }
  p
}
```

## Do not

- Do not **`file.copy`** exploratory PNG/PDF as the **canonical** composite — **regenerate** with R after layout or data changes.
- Do not treat **non-rectangular** letter regions as one bbox without **changing `design.txt`** — the rectangle assert must pass.
