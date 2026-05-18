---
name: manuscript-figure-curation
description: >-
  Curates publication figures from a messy analysis repo into a clean manuscript
  tree: locates image outputs, traces minimal R code and yaml inputs, copies or
  re-exports assets, and updates figure_manifest.yaml. **Curation (收录)** keeps the
  upstream figure’s **non-text look** (borders, spacing, colours, axis geometry); on
  top of that, **align font family and point sizes** to `.cursorrules` **§2C** — **Mode 2**
  by default for **all** figures; **Mode 1** only if the user
  **explicitly** requests it (chat or yaml). Full minimalist **§2C** theme replacement is
  **only** for **重画刊例** / **fig-spec refresh**. Each main figure uses an **F1** / **F2** /
  **FS*** bundle under MANUSCRIPT_ROOT. Run with conda R; exports follow §3 (ggsave,
  scale, dpi, cairo_pdf).
  **Whole-figure `design.txt` assembly** uses the companion skill **`manuscript-composite-layout`** (NPC `draw_plot` matrix method).
  **Volcano + heatmap:** use the **embedded reference R** and style rules in the body of this skill (**self-contained** — do not depend on other repo files to define the look).
  Use when the user assigns a panel, harvests plots, updates **`figure_manifest.yaml`**, or uses 重画刊例 / fig-spec refresh; use **both** skills when the task includes full-figure layout. **Export stems** default to **`Fig0N_panel_X`**: two-digit
  main figure index **N** (digits only), panel **X** as a **single uppercase letter** (A–Z);
  when assignment is unspecified or the user asks for **顺延**, continue from the highest
  existing main index **N** / panel letter **X** in the manuscript tree.

---
# Manuscript figure curation

## When this applies

The user (or conversation) specifies **which panel is for the article**, using their journal naming, for example:

- "Figure 2 panel B" / "Fig. 2B" / "Figure S3 A"
- Or a **single file** that already is the final composite for that label

If **panel X** is not a separate file (e.g. one PNG contains 2×2 panels), treat **one file** as the whole Figure N unless the user points to a crop or a separate export; **ask once** if it is unclear whether they want the full composite or a single panel.

## Manuscript root (per project)

The **clean directory** is not global: it can differ per repository. Resolve the root **in this order**:

1. **Explicit path in the user message** for this task (e.g. “输出到 `/path/to/doc2/`”) — wins for that request only.
2. **Workspace file** (recommended): at the **git / analysis project root**, a single line, absolute path, no quotes, no trailing spaces:
   - **Primary:** `.manuscript_curate_root`
   - **Alternate:** `.cursor/manuscript_curate_root`  
   Read the first line only; ignore blank lines and `#` comments.
3. If **none** of the above exist, **ask the user** for the absolute path once, then offer to create `.manuscript_curate_root` so the next session does not need repeating.

Call this resolved directory **`MANUSCRIPT_ROOT`**. Curated paths are under **`MANUSCRIPT_ROOT`**, not necessarily under the exploratory repo root.

**User operations (per project):**

- **Set this project:** create or edit `.manuscript_curate_root` in that project’s root and put one line, e.g. `/home/likun/likun/99.likun/doc/doc1`
- **Switch to another project:** open the other repo; there either create a different `.manuscript_curate_root` or rely on an explicit path in chat. No need to edit the skill.

## Figure bundle: one folder per main figure (required)

**Rule:** For each **journal main figure** (正文图、补充图各自算一张「主图」), keep **all** curated assets in **one** directory: final PNG/PDF (and optional SVG), minimal R scripts, and the yaml / small tables those scripts read. Do **not** split panels of the same figure across unrelated top-level folders.

**Bundle directory names** (choose **one** convention per project and stick to it):

- Main text: e.g. `F1`, `F2`, `F3` or `Fig01`, `Fig02`.
- Supplementary: e.g. `FS1`, `FS2` or `FigS1`, `FigS2`.

Map labels clearly: Figure 1 → `F1`; Figure S1 → `FS1`. If the user already uses another pattern, follow it and record it in **`figure_manifest.yaml`** (optional top-level note such as `bundle_naming`).

**Inside each bundle**

- **`figures/`** — exports for **that** figure only (panels and optional full composite).
- **`scripts/`** — minimal R for **that** figure (per-panel scripts, composite, optional local `_style.R`).
- **`config/`** — yaml and tiny tables needed to rerun those scripts (paths documented relative to bundle or `MANUSCRIPT_ROOT`).

Optional **shared** helpers (`MANUSCRIPT_ROOT/scripts/_style.R`, `MANUSCRIPT_ROOT/config/reproduction_lineage.yaml`) are allowed when intentional; **figure-specific scripts and exports still live under that figure’s bundle**.

**Legacy flat trees:** Older layouts with top-level `MANUSCRIPT_ROOT/figures/` and `scripts/` should migrate on refresh: move into the correct **`F*`** / **`FS*`** bundle and update **`figure_manifest.yaml`**.

## Preserve source style on curation (收录)

**Default policy (two layers):**

1. **Keep upstream “look” (样式 / geometry)** — Reuse the exploratory plot’s **non-typography** choices so the curated figure still **reads like the same figure**: `theme()` / `scale_*` / colours / `panel.border`, `panel.spacing`, `axis.line`, strip boxes, legend **position**, margins, facet layout, and geoms stay aligned with the **canonical source** unless the user asks to change them.
2. **Then align typography (字体 + 字号)** — On that base, **do** apply **`family = "Arial"`** and **`.cursorrules` §2C** sizes: **Mode 2** unless the user **explicitly** asked for **Mode 1** (same rule for standalone and composite — do **not** shrink to Mode 1 just because panels are combined). Implement with **`+ theme(...)` additions** on each ggplot panel that **only** touch **text** elements and explicit `size` / `family` on text geoms — **not** by replacing the whole theme with a packaged full theme that changes panel geometry unless the user asked for **full restyle**.

**Problem to avoid:** Applying a **full** journal theme helper (e.g. a packaged `fig02_mode1_theme()` or a copy of the entire `.cursorrules` global `theme_set(...)` block) during **curation** when the user only wanted **font + size + export** compliance. That can silently change **layout geometry** (`panel.border`, `panel.spacing.x` / `panel.spacing.y`, `axis.line`, `strip.background`, margins, legend position, etc.) and make the curated output look unlike the **canonical exploratory figure**.

**Rules:**

- **Curation (收录)** = minimal traced code + **layer (1)** + **layer (2)** above + paths, bundle layout, and **§3** export. If “moving toward `.cursorrules`” would require changing non-text theme elements, **stop** and **ask** the user or treat it as **full restyle** (below).
- **Full restyle** = **only** when the user explicitly asks for **`重画刊例`**, **`按刊例重画`**, **`fig-spec refresh`**, **`/fig-spec-refresh`**, or clearly states they want the **complete** minimalist look from `.cursorrules` (whole theme block). Then a complete theme is allowed; document in **`figure_manifest.yaml`** (e.g. `style: cursorrules_full`).
- **Never** “fix” borders, spacing, or axis lines during curation for “consistency” alone.
- **Sanity check:** After the first curated render, compare to the **source** PNG/PDF. The figure should match apart from **font family, font sizes,** and intentional export dimensions / resolution. If **geometry** diverges, treat it as a mistake unless **full restyle** was requested.

## Target directory layout (under `MANUSCRIPT_ROOT`)

```text
figure_manifest.yaml       # index; paths reference bundle subfolders
README.md                  # conda env, run order; optional if user declines md
config/                    # optional: shared lineage, global small tables
scripts/                   # optional: shared _style.R or utilities

F1/                        # Figure 1 bundle (example)
  figures/
  scripts/
  config/

F2/
  figures/
  scripts/
  config/

FS1/                       # supplementary Figure S1 (example)
  figures/
  scripts/
  config/
```

**Naming inside a bundle — required export stem (default)**

- **Per-panel raster / vector exports** use this stem (English, no journal-specific punctuation in the filename):
  - **`Fig0N_panel_X`** (read as: **`Fig`** + **two-digit main index** + **`_panel_`** + **panel letter**).
  - **`N`** = **main text figure index** as **two digits** (`01`, `02`, `03`, …) — **digits only** in the `Fig` prefix segment. **`X`** = **panel designation** as a **single uppercase letter** (`A`, `B`, `C`, … through `Z`). **Do not** use numeric panel suffixes in the filename for new work.
  - **Examples:** `Fig01_panel_A.png`, `Fig01_panel_A.pdf`, `Fig03_panel_B.pdf`.
  - **Same stem** for `.png` and `.pdf` whenever both exist. **Scripts** mirror the stem: e.g. `F3/scripts/Fig03_panel_B.R`.
- **Journal label vs file:** The manuscript may say “Fig. 2B”; **`figure_manifest.yaml`** `label` keeps that wording. **`outputs`** paths use the **same letter** as the journal panel when they align (e.g. **`Fig02_panel_B`** for “Fig. 2B”). If journal letter order differs from file ordering, document the mapping in the manifest.
- **Whole-figure composite:** prefer **`Fig0N_composite_drawplot`** (`F*/scripts/Fig0N_composite_drawplot.R`, `F*/figures/Fig0N_composite_drawplot.png|pdf`) with **`design.txt`** per **`manuscript-composite-layout`**; register in **`figure_manifest.yaml`**. Older composite stems may remain until retired.
- **Supplementary figures:** Prefer **`FigS0N_panel_X`** with the same rule (**`N`** two-digit supplement main index, **`X`** uppercase letter); record in **`figure_manifest.yaml`** (`bundle_naming`, `figure_stem_pattern`) if you use another single project-wide pattern.
- **Manifest id:** Stable snake_case id derived from the stem, e.g. `fig03_panel_b` or `fig03_pb` — **one convention per project**; stay consistent.

**When main index `N` and panel letter `X` are not given, or the user asks for 顺延 (continue in sequence)**

1. **Scan authoritative sources** under **`MANUSCRIPT_ROOT`**: `figure_manifest.yaml` **`outputs`** paths, existing **`F*/figures/`** and **`FS*/figures/`** files matching `Fig0*_panel_[A-Z]` / `FigS*_panel_[A-Z]`, and bundle folder names. Take the **maximum** numeric **main index `N`** (and **supplement `S` index** if applicable) already in use.
2. **New main figure** (user adds “another main figure” but does **not** assign a journal figure number): assign **`N = max(N) + 1`** (two digits), and start panels at **`X = A`** (first export: `Fig0N_panel_A`).
3. **New panel under an existing main figure `N`** (user does **not** specify the panel letter, or says **顺延** for “next panel”): set **`X`** to the **uppercase letter immediately after** the **highest panel letter already used** for that same **`N`** (same bundle), using **A–Z** order (e.g. existing `A` and `C` → next default **`D`**). If nothing exists yet for that **`N`**, use **`X = A`**.
4. **User explicitly gives figure and panel** (e.g. “正文图 3 的 B panel” / “Fig. 2C”): use that **`N`** and **`X`**; **do not** auto-shift unless the user asks to renumber or **顺延** from a stated anchor.
5. **Ambiguity** (two bundles both claim `Fig03`, or manifest vs disk disagree): **ask once**, then fix manifest and filenames together.

**Legacy:** Older stems with **numeric** panel suffixes (e.g. `Fig02_panel_1`) should be renamed on the next curation pass to **`Fig02_panel_A`** (or the letter that matches the journal), unless the project manifest explicitly keeps numeric panels (document under `figure_stem_pattern: legacy_numeric_panel`).

## Volcano plots — style + **embedded reference R** (no external exemplar files)

Use **`ggplot2` + `ggpubr`**; on-figure text **English only**; **`family = "Arial"`**; legend **bottom**; **Flat UI** point colours: **`#95A5A6`** (Not sign), **`#E74C3C`** (Up), **`#3498DB`** (Down); **`geom_point(..., stroke = 0)`**; **`scale_alpha_identity()`** when mapping alpha in `aes()`; **`theme_pubr(base_size = 9, border = TRUE)`** plus **§2C Mode 2** text sizes on **收录** (full **§2C** `theme_set` only under **`重画刊例`** / **`fig-spec refresh`**). **`ggsave`**: default **`.cursorrules` §3** (`width = 7`, `height = 5`, `scale = 1.3`, `dpi = 300`, PDF **`device = cairo_pdf`**).

**Two standard variants** (adapt column names / yaml thresholds to the project; wire into **`build_<manifest_id>(root)`** returning **`ggplot`**):

### A — General DE volcano (raw *P*, optional x cap for display)

- **y** = **−log₁₀(raw *p*)**; classify **Up / Down / Not sign** from **raw *p*** (e.g. `< 0.05`) and sign of log2FC.
- Optional: **`log2fc_plot <- pmax(pmin(log2fc, cap), -cap)`** + **`coord_cartesian(xlim = c(-cap, cap))`** when the paper uses a narrow x window.
- **`geom_hline(-log10(0.05), linetype = "dashed")`**; **`geom_vline(0, ...)`**.

```r
# Reference: general DE volcano — expects columns: log2fc, p_value (and optional gene id, not plotted)
manuscript_volcano_de_ggplot <- function(
    d,
    x_cap = 0.15,
    p_cut = 0.05,
    plot_title = "Volcano",
    x_label = "Log2 fold change (after / before)",
    y_label = "-Log10 P-value"
) {
    stopifnot(is.data.frame(d), all(c("log2fc", "p_value") %in% names(d)))
    library(ggplot2)
    library(ggpubr)
    vd <- d[!is.na(d$p_value), , drop = FALSE]
    vd$log10p <- -log10(vd$p_value)
    vd$significant <- vd$p_value < p_cut
    vd$direction <- ifelse(
        vd$significant & vd$log2fc > 0, "Up",
        ifelse(vd$significant & vd$log2fc < 0, "Down", "Not sign")
    )
    vd$direction <- factor(vd$direction, levels = c("Not sign", "Up", "Down"))
    vd$log2fc_plot <- pmax(pmin(vd$log2fc, x_cap), -x_cap)
    ggplot(
        vd,
        aes(
            x = log2fc_plot, y = log10p, color = direction,
            alpha = ifelse(significant, 0.8, 0.3)
        )
    ) +
        geom_point(size = 1.1, stroke = 0) +
        scale_color_manual(
            values = c("Not sign" = "#95A5A6", "Up" = "#E74C3C", "Down" = "#3498DB"),
            name = NULL, breaks = c("Not sign", "Up", "Down")
        ) +
        scale_alpha_identity() +
        coord_cartesian(xlim = c(-x_cap, x_cap), clip = "on") +
        scale_y_continuous(expand = expansion(mult = c(0.02, 0.04))) +
        geom_hline(yintercept = -log10(p_cut), linetype = "dashed", color = "black", alpha = 0.5) +
        geom_vline(xintercept = 0, linetype = "dashed", color = "black", alpha = 0.3) +
        labs(title = plot_title, x = x_label, y = y_label) +
        guides(color = guide_legend(nrow = 1, title.position = "top", byrow = TRUE)) +
        theme_pubr(base_size = 9, border = TRUE) +
        theme(
            text = element_text(family = "Arial"),
            plot.title = element_text(hjust = 0.5, size = 11, face = "plain", family = "Arial"),
            plot.subtitle = element_blank(),
            plot.caption = element_blank(),
            axis.title = element_text(size = 9, family = "Arial"),
            axis.title.y = element_text(margin = margin(r = 3, unit = "pt"), family = "Arial"),
            axis.text = element_text(size = 8, color = "#2C3E50", family = "Arial"),
            legend.position = "bottom",
            legend.direction = "horizontal",
            legend.box.margin = margin(-6, 0, 0, 0, "pt"),
            legend.margin = margin(-2, 0, 0, 0, "pt"),
            legend.title = element_text(size = 8, family = "Arial"),
            legend.text = element_text(size = 6.5, family = "Arial"),
            plot.margin = margin(3, 4, 0, 4, "pt"),
            panel.background = element_rect(fill = "white", color = NA),
            panel.border = element_rect(color = "black", fill = NA, linewidth = 0.5),
            axis.line = element_blank(),
            axis.ticks = element_line(linewidth = 0.3)
        )
}
```

### B — Site-level / limma volcano (raw *P* on y; colour by BH-FDR)

- **y** = **−log₁₀(raw *p*)** with **`pmax(p, .Machine$double.xmin)`** before log.
- **Colour** by **BH-FDR** vs **`fdr_threshold`** from yaml (`.cursorrules` Griffin / cfDNA style).
- **Vertical lines:** **x = 0**; **x = ±** **`fc_thr`** with **`fc_thr <- median(abs(logFC), na.rm = TRUE) * 1.5`** (tune multiplier in yaml if needed).
- **y-axis label:** use **`expression(-log[10] ~ "(" * plain(raw) ~ italic(P) * ")")`**.

```r
# Reference: limma-style volcano — expects columns: limma_logFC, limma_p_value, limma_p_adjust
manuscript_volcano_limma_fdr_ggplot <- function(
    d,
    fdr_threshold = 0.2,
    plot_title = "Volcano",
    x_label = "log2 fold change (after vs before)"
) {
    stopifnot(
        is.data.frame(d),
        all(c("limma_logFC", "limma_p_value", "limma_p_adjust") %in% names(d))
    )
    library(ggplot2)
    library(ggpubr)
    vd <- d[!is.na(d$limma_p_value) & !is.na(d$limma_p_adjust), , drop = FALSE]
    vd$log10p <- -log10(pmax(vd$limma_p_value, .Machine$double.xmin))
    vd$sig_fdr <- vd$limma_p_adjust < fdr_threshold
    vd$point_col <- "Not sign"
    vd$point_col[vd$sig_fdr & vd$limma_logFC > 0] <- "Up"
    vd$point_col[vd$sig_fdr & vd$limma_logFC < 0] <- "Down"
    vd$point_col <- factor(vd$point_col, levels = c("Not sign", "Up", "Down"))
    fc_thr <- stats::median(abs(vd$limma_logFC), na.rm = TRUE) * 1.5
    ggplot(
        vd,
        aes(
            x = limma_logFC, y = log10p, color = point_col,
            alpha = ifelse(point_col != "Not sign", 0.85, 0.35)
        )
    ) +
        geom_point(size = 1.15, stroke = 0) +
        scale_color_manual(
            values = c("Not sign" = "#95A5A6", "Up" = "#E74C3C", "Down" = "#3498DB"),
            name = NULL, breaks = c("Not sign", "Up", "Down")
        ) +
        scale_alpha_identity() +
        geom_vline(xintercept = 0, linetype = "dashed", color = "#34495E", linewidth = 0.35, alpha = 0.5) +
        geom_vline(xintercept = c(-fc_thr, fc_thr), linetype = "dashed", color = "#95A5A6", linewidth = 0.25, alpha = 0.6) +
        scale_x_continuous(expand = expansion(mult = c(0.02, 0.02))) +
        scale_y_continuous(expand = expansion(mult = c(0.02, 0.05))) +
        labs(
            title = plot_title, x = x_label,
            y = expression(-log[10] ~ "(" * plain(raw) ~ italic(P) * ")")
        ) +
        guides(color = guide_legend(nrow = 1, title.position = "top", byrow = TRUE)) +
        theme_pubr(base_size = 9, border = TRUE) +
        theme(
            text = element_text(family = "Arial"),
            plot.title = element_text(hjust = 0.5, size = 11, face = "plain", family = "Arial"),
            plot.subtitle = element_blank(),
            plot.caption = element_blank(),
            axis.title = element_text(size = 9, family = "Arial"),
            axis.text = element_text(size = 8, color = "#2C3E50", family = "Arial"),
            legend.position = "bottom",
            legend.direction = "horizontal",
            legend.box.margin = margin(-6, 0, 0, 0, "pt"),
            legend.margin = margin(-2, 0, 0, 0, "pt"),
            legend.title = element_text(size = 8, family = "Arial"),
            legend.text = element_text(size = 7.5, family = "Arial"),
            plot.margin = margin(3, 4, 0, 4, "pt"),
            panel.background = element_rect(fill = "white", color = NA),
            panel.border = element_rect(color = "black", fill = NA, linewidth = 0.5),
            axis.line = element_blank(),
            axis.ticks = element_line(linewidth = 0.3)
        )
}
```

**Implementation rules:** read inputs from **`<bundle>/config/*.yaml`**; never **`file.copy`** exploratory PNG as the final panel; **`build_<manifest_id>(root)`** returns **`ggplot`**.

## Heatmaps — **ggplot2** body (style + **embedded reference R**)

**Deliverable** = one **`ggplot`**: **row-clustered** matrix (features × samples), **z-score clipped** (e.g. ±2), **`geom_tile`** heat body, optional **sample annotation** strip (**`geom_tile`** + **`scale_fill_manual`**), **left row dendrogram** as **`geom_segment`**, **`ggnewscale::new_scale_fill()`** between annotation and heat fills. **`theme_pubr(..., border = FALSE)`**, no axis text/ticks (matrix layout), **`coord_cartesian(clip = "off")`**, legends **bottom** horizontal. **Packages:** `ggplot2`, `ggpubr`, `ggnewscale`, `dplyr`, `tidyr`, `tibble`, `grid`.

**`sample_group`** = one character/factor per column of **`M`** (e.g. `after` / `before`); factor **levels** in the strip should match **`scale_fill_manual`** (example uses **`after`**, **`before`** with **`#E74C3C`** / **`#3498DB`**).

```r
# Row dendrogram segments for ggplot (from stats::hclust merge matrix)
manuscript_hclust_segments <- function(hc) {
    n <- length(hc$order)
    leaf_y <- integer(n)
    leaf_y[hc$order] <- seq_len(n)
    node_y <- numeric(n - 1L)
    node_h <- hc$height
    segs <- vector("list", 3L * (n - 1L))
    k <- 0L
    get_child <- function(idx) {
        if (idx < 0L) {
            leaf <- -idx
            list(y = leaf_y[[leaf]], h = 0)
        } else {
            list(y = node_y[[idx]], h = node_h[[idx]])
        }
    }
    for (i in seq_len(n - 1L)) {
        left <- get_child(hc$merge[i, 1])
        right <- get_child(hc$merge[i, 2])
        node_y[[i]] <- mean(c(left$y, right$y))
        h <- node_h[[i]]
        k <- k + 1L
        segs[[k]] <- data.frame(x = left$h, xend = h, y = left$y, yend = left$y)
        k <- k + 1L
        segs[[k]] <- data.frame(x = right$h, xend = h, y = right$y, yend = right$y)
        k <- k + 1L
        segs[[k]] <- data.frame(x = h, xend = h, y = left$y, yend = right$y)
    }
    dplyr::bind_rows(segs)
}

# Matrix heatmap: rows = features, columns = samples; values typically row z-scores
manuscript_matrix_heatmap_ggplot <- function(M, sample_group, plot_title, dend_width_mult = 1) {
    if (!requireNamespace("ggplot2", quietly = TRUE)) stop("Need ggplot2")
    if (!requireNamespace("ggpubr", quietly = TRUE)) stop("Need ggpubr")
    if (!requireNamespace("ggnewscale", quietly = TRUE)) stop("Need ggnewscale")
    if (!requireNamespace("dplyr", quietly = TRUE)) stop("Need dplyr")
    if (!requireNamespace("tidyr", quietly = TRUE)) stop("Need tidyr")
    if (!requireNamespace("tibble", quietly = TRUE)) stop("Need tibble")
    stopifnot(length(dend_width_mult) == 1L, is.finite(dend_width_mult), dend_width_mult > 0)
    M <- as.matrix(M)
    sample_group <- as.character(sample_group)
    stopifnot(length(sample_group) == ncol(M), !anyNA(sample_group))
    hc <- stats::hclust(stats::dist(M))
    row_order <- hc$order
    M <- M[row_order, , drop = FALSE]
    M_plot <- pmax(pmin(M, 2), -2)
    n_row <- nrow(M_plot)
    n_col <- ncol(M_plot)
    anno_h <- max(8, n_row * 0.055)
    gap_y <- max(2, n_row * 0.012)
    dend_w <- max(3.5, n_col * 0.12) * dend_width_mult
    heat_df <- as.data.frame(M_plot, check.names = FALSE) |>
        tibble::rownames_to_column("feature") |>
        dplyr::mutate(row_index = dplyr::row_number()) |>
        tidyr::pivot_longer(cols = -c(feature, row_index), names_to = "sample", values_to = "z") |>
        dplyr::mutate(sample_index = match(.data$sample, colnames(M_plot)))
    anno_df <- data.frame(
        sample_index = seq_len(n_col),
        anno_y = -gap_y - anno_h / 2,
        Time = factor(sample_group, levels = c("after", "before"))
    )
    dend_df <- manuscript_hclust_segments(hc)
    max_h <- max(dend_df$x, dend_df$xend, na.rm = TRUE)
    if (!is.finite(max_h) || max_h <= 0) max_h <- 1
    dend_df <- dend_df |>
        dplyr::mutate(
            x = -(.data$x / max_h) * dend_w,
            xend = -(.data$xend / max_h) * dend_w
        ) |>
        dplyr::filter(!(.data$x == -dend_w & .data$xend == -dend_w))
    ggplot2::ggplot() +
        ggplot2::geom_segment(
            data = dend_df,
            ggplot2::aes(x = .data$x, xend = .data$xend, y = .data$y, yend = .data$yend),
            linewidth = 0.08, colour = "black", alpha = 0.85
        ) +
        ggplot2::geom_tile(
            data = anno_df,
            ggplot2::aes(x = .data$sample_index, y = .data$anno_y, fill = .data$Time),
            width = 1, height = anno_h
        ) +
        ggplot2::scale_fill_manual(
            values = c("after" = "#E74C3C", "before" = "#3498DB"),
            name = "Time",
            guide = ggplot2::guide_legend(
                title.position = "top", order = 2,
                keywidth = grid::unit(2.4, "mm"), keyheight = grid::unit(2.4, "mm")
            )
        ) +
        ggnewscale::new_scale_fill() +
        ggplot2::geom_tile(
            data = heat_df,
            ggplot2::aes(x = .data$sample_index, y = .data$row_index, fill = .data$z),
            width = 1, height = 1
        ) +
        ggplot2::scale_fill_gradient2(
            low = "blue", mid = "white", high = "red", midpoint = 0, limits = c(-2, 2),
            name = "Z-score",
            guide = ggplot2::guide_colorbar(
                title.position = "top", order = 1,
                barwidth = grid::unit(12, "mm"), barheight = grid::unit(2, "mm")
            )
        ) +
        ggplot2::scale_x_continuous(expand = c(0, 0), limits = c(-dend_w, n_col + 0.5)) +
        ggplot2::scale_y_reverse(expand = c(0, 0), limits = c(n_row + 0.5, -anno_h - gap_y)) +
        ggplot2::coord_cartesian(clip = "off") +
        ggplot2::labs(title = plot_title, x = NULL, y = NULL) +
        ggpubr::theme_pubr(base_size = 9, border = FALSE) +
        ggplot2::theme(
            text = ggplot2::element_text(family = "Arial"),
            plot.title = ggplot2::element_text(hjust = 0.5, size = 11, face = "plain", family = "Arial"),
            axis.title = ggplot2::element_blank(),
            axis.text = ggplot2::element_blank(),
            axis.ticks = ggplot2::element_blank(),
            axis.line = ggplot2::element_blank(),
            panel.border = ggplot2::element_blank(),
            panel.background = ggplot2::element_rect(fill = "white", color = NA),
            legend.title = ggplot2::element_text(size = 8, family = "Arial"),
            legend.text = ggplot2::element_text(size = 7.5, family = "Arial"),
            legend.position = "bottom",
            legend.box = "horizontal",
            legend.box.margin = ggplot2::margin(-6, 0, 0, 0, "pt"),
            legend.margin = ggplot2::margin(-2, 0, 0, 0, "pt"),
            legend.key.size = grid::unit(2.4, "mm"),
            plot.margin = ggplot2::margin(0, 0, 0, 0)
        )
}
```

**Upstream:** read matrix + grouping from yaml; **ComplexHeatmap** may exist in pipelines for exploration — **curated** panel should still be this **ggplot2** object unless the user explicitly requires CH-only export. **`build_*`** returns **`ggplot`**; **`ggsave`** per **§3**.

## Workflow (agent checklist)

1. **Resolve the label and bundle**  
   Map "Figure N, panel …" to a **manifest id**, file stem **`Fig0N_panel_X`** (two-digit **`N`**, uppercase letter **`X`**, or explicit user override), and **bundle** (e.g. `F2` for Figure 2, `FS1` for Figure S1). If the user **does not** specify **`N`/`X`** or asks for **顺延**, apply the **“When main index `N` and panel letter `X` are not given”** rules above before creating files. Create `figures/`, `scripts/`, `config/` under that bundle if missing.

2. **Find the image**  
   Search the workspace for likely outputs: `rg` / IDE search for `ggsave`, `png(`, `pdf(`, `cairo_pdf`, `ComplexHeatmap::`, `saveRDS` near plot code; match **filename fragments** the user remembers (path, prefix, analysis step).  
   If multiple candidates: list the **top few paths** and prefer the one that matches **date, pipeline folder, or log**; if still ambiguous, **ask the user** which path is canonical.

3. **Find the generating code**  
   From the winning image path, locate the **R script** that wrote it (same directory logs, `ggsave` string match, or git history). Read enough context to know **inputs** (RDS, CSV, expression matrix) and **yaml** keys.

4. **Minimal reproducible script**  
   Create **`MANUSCRIPT_ROOT/<bundle>/scripts/Fig0N_panel_X.R`** (two-digit **`N`**, uppercase letter **`X`**) with the **composite-ready** pattern below and **exports** that follow **`.cursorrules` §3** (`ggsave` width/height, **`scale = 1.3`**, **`dpi = 300`**, PDF **`device = cairo_pdf`**).  
   - **Style (see “Preserve source style” above):** On **curation (收录)**, keep upstream **geometry and non-text theme**; add **typography only** — **Arial** and **§2C Mode 2** pt sizes by default (or **Mode 1** if the user explicitly requested it), via `+ theme()` / text-geom `size`, **without** swapping in a full packaged theme that changes `panel.border`, `panel.spacing`, `axis.line`, strips, or legend **placement**. Full **§2C** body restyle is for **`重画刊例`** / **`fig-spec refresh`** when intentional; then set **`style: cursorrules_full`** in **`figure_manifest.yaml`**.  
   - **Composite-ready interface (required for ggplot panels):** define **`build_<manifest_id>(root)`** (e.g. **`build_fig03_panel_b(root)`**) that **returns** the **`ggplot` object** and performs **no** `ggsave`. Place **`ggsave` only in a bottom block** that runs **only when this file is the `Rscript` entry** (e.g. `commandArgs` has `--file=` **and** its normalized path **ends with this script’s basename**). Whole-figure assembly from **`design.txt`** follows **`manuscript-composite-layout`** (named list of `build_*` → **`ggdraw` + `draw_plot`**). One **`ggsave`** at **180 mm** (or project default) for the composite script entry (§3). **Do not** rely on large **`saveRDS(ggplot)`** as the primary handoff unless the user explicitly wants it.  
   - **`root`:** use **`MANUSCRIPT_ROOT`** or the **bundle path** consistently; state at the top of the script how `config/` paths resolve.  
   - Loads only libraries and functions needed for **that** plot  
   - Reads inputs via **`<bundle>/config/`** or documented shared `MANUSCRIPT_ROOT/config/` — avoid hard-coded exploratory paths when a **copy into `<bundle>/config/`** is reasonable  
   - English on-figure text; for **new** color choices (not present upstream), prefer **Flat UI** hex from `.cursorrules`  
   - **Does not** use `tryCatch` or other error masking unless the user explicitly overrides project rules  
   - **Legacy** one-file scripts that `stop()` unless launched via `Rscript`: refactor to the export-guard pattern the **first time** they must participate in an in-R composite.

5. **Run script immediately**  
   After the script exists, run **`conda activate R`** then **`Rscript`** with working directory **`MANUSCRIPT_ROOT`** or the **bundle** root (**pick one** and match the script). Writes go to **`<bundle>/figures/`** in the **same** curation session. Perform the **sanity check** (visual match to source) when the task is **收录**, not only full **重画刊例**.  
   - **ggplot / cowplot**: outputs must be generated by this run, not copied from old `res/` at wrong size.  
   - **Heatmaps:** prefer a **ggplot2** panel using **`manuscript_matrix_heatmap_ggplot()`** (or equivalent) from **「Heatmaps — ggplot2 body」** in this skill. If upstream is **raster-only** (e.g. **`png()`** from **ComplexHeatmap** in the exploratory repo), **rebuild** as ggplot2 when feasible; otherwise **rerun** upstream with vector-friendly settings, then document in **`figure_manifest.yaml`** if the tree still depends on an external script.  
   - If rerun is **not** feasible in one session, set **`status: needs_redraw`** in the manifest and state what is missing.

6. **Assets**  
   - **Default**: canonical outputs under **`<bundle>/figures/`**. **Do not** copy an exploratory PNG/PDF over the freshly generated file unless the user explicitly asks for a **frozen snapshot** of an old render.  
   - Prefer **both** `.png` and `.pdf` whenever the plot is ggplot-based (including **ggplot2 heatmaps**); if only exploratory raster exists and ggplot rebuild is pending, **pdf** may be absent — record in manifest.

7. **Config**  
   Copy or slim **yaml** (and tiny sample/group tables) into **`<bundle>/config/`**. Use **`MANUSCRIPT_ROOT/config/`** only for **cross-figure** shared files. If full data cannot be copied, record in manifest: **data location + version + how obtained**.

8. **Update `MANUSCRIPT_ROOT/figure_manifest.yaml`**  
   Append or update one entry per panel or composite; each entry must include **`bundle`** and paths under that bundle (see example).

   ```yaml
   figures:
     fig02_panel_b:
       label: "Figure 2B"
       bundle: F2
       script: F2/scripts/Fig02_panel_B.R
       builder: build_fig02_panel_b   # R function name returning ggplot object
       outputs:
         png: F2/figures/Fig02_panel_B.png
         pdf: F2/figures/Fig02_panel_B.pdf
       inputs:
         config: F2/config/analysis_fig02_panel_b.yaml
       source_exploratory: "optional original path + script for traceability"
       style: source_matched   # default: same geometry as source + Arial + §2C Mode 2 (Mode 1 only if user explicit)
       status: draft   # draft | final | needs_redraw
   ```

9. **Reply to the user**  
   In Chinese if they write in Chinese: short summary, **absolute paths** of new script, yaml, and figure files (include **bundle** path).

## Shorthand: **重画刊例** / **fig-spec refresh**

If the user writes **`重画刊例`**, **`按刊例重画`**, or **`fig-spec refresh`** with one or more paths, apply the same intent as **`.cursorrules` §5** (refresh plots per **§2C + §3**, run with **`conda activate R`**, keep output paths unless overridden). The same workflow applies when the user runs the slash command **`/fig-spec-refresh`** (see **`.cursor/commands/fig-spec-refresh.md`**). If they also ask to refresh curated assets, update **`MANUSCRIPT_ROOT`**, the relevant **bundle** under **`F*`** / **`FS*`**, and **`figure_manifest.yaml`**.

**Relationship to curation:** Steps **4–5** rerun the curated script; **收录** = **preserve source geometry and style**, **plus** **font family + §2C Mode 2** sizes (unless the user explicitly asked for **Mode 1**). Use **`/fig-spec-refresh`** / **`重画刊例`** for deliberate full minimalist styling or after **`.cursorrules`** / data / script changes.

## Clarifications to ask (only if needed)

- **Which file** is Figure N X if several versions exist (dates, parameters).
- **Bundle folder naming** if `F1` vs `Fig01` (or similar) is ambiguous — align once with the user.
- **Data policy**: whether intermediate RDS may be copied into `manuscript/` or must remain read-only elsewhere.
- **README**: omit if the user forbids new markdown files; otherwise one short `MANUSCRIPT_ROOT/README.md` with `conda activate R` and run order is enough.

## What “minimal” means

- **Minimal** = smallest set of code and config that **rebuilds that panel** from documented inputs, not a copy of entire pipelines.  
- It is acceptable to **factor** repeated style setup into **`MANUSCRIPT_ROOT/scripts/_style.R`** **only if** the user already uses that pattern or agrees to reduce duplication; **exports stay** in **`<bundle>/figures/`**.

## Multi-panel composites (whole figures)

Authoritative **layout + assembly** for **`design.txt`** lives in the companion skill **`manuscript-composite-layout`**: character **cell matrix** → **NPC** bounding boxes → **`cowplot::ggdraw` + `draw_plot`** (+ **`draw_label`** / optional tag remap). Per-panel **`build_*`** functions, yaml, and **`figure_manifest.yaml`** stay in **this** skill; register composite scripts and outputs in the manifest alongside panels.

## Composite export note (Illustrator)

- **Width / typography:** follow project **`.cursorrules`** (§2C / §3); full-width statistical composites are commonly **`width = 180 / 25.4`** in, PNG **`dpi = 300`**, PDF **`device = cairo_pdf`**; **`scale`** must match the bundle’s chosen composite policy (see **manuscript-composite-layout**).
- **Hybrid:** export the **R** statistical block as **PDF**, then combine with mechanism art in **Illustrator** at uniform scale.

## Data provenance (journal-ready repository)

Goal: every curated figure can be traced **from raw sequencing inputs** (or their **public accessions**) through **Snakemake (or equivalent)** → **quantified matrix** → **differential analysis** → **downstream statistics / enrichment** → **ggplot** (volcano, **ggplot2 heatmaps**, bars, etc.) in **`MANUSCRIPT_ROOT/<bundle>/scripts/`**, so a reviewer can reproduce or audit the chain. **Heatmap panels** default to the **ggplot2** pattern in **「Heatmaps — ggplot2 body」**; **ComplexHeatmap** may appear upstream or for legacy validation — the **curated** export should still be **ggplot2** when this skill applies unless the user opts out. Any **`F*`** / **`FS*`** bundle may host the first volcano or heatmap script.

**Tiers (document in `MANUSCRIPT_ROOT/config/reproduction_lineage.yaml` and keep `figure_manifest.yaml` in sync):**

1. **Raw data** — FASTQ/BAM are usually **not** in git; list **SRA/EGA/dbGaP accessions**, ethics statement if needed, and where controlled-access files live for your lab.
2. **Alignment / quantification** — `Snakefile` (or workflow URL + commit), conda/singularity profile, rule outputs that produce the **gene-level matrix** (FPKM/counts) and any **fragment-level** files (e.g. Griffin) consumed by later R steps.
3. **Differential expression** — script + yaml that reads the matrix and writes **DE tables** (CSV/RDS) used by volcano / heatmap / gene lists for enrichment.
4. **Enrichment (GO/KEGG)** — must depend on the **same DE-derived gene list** as the manuscript text. **Preferred:** upstream pipeline saves **`enrichResult` RDS** (`ego_all`, `kegg_result`, …); curated scripts call **`enrichplot::dotplot()`** (or equivalent) and keep **upstream styling** on **收录**; apply the **full** `.cursorrules` theme only under explicit **`重画刊例`** / **`fig-spec refresh`** — **not** `readRDS(ggplot)` as the only scientific input. **Interim:** if only **ggplot RDS** exists, manifest **must** record the **exact upstream script** (e.g. `34...R.06.GO_KEGG.R`) and **CSV exports** (`GO_enrichment_ALL.csv`, `KEGG_enrichment.csv`) as provenance; plan to add **`saveRDS(enrichResult)`** in the exploratory pipeline so the curated layer can drop ggplot-only intermediates.
5. **Heatmaps** — inputs = **DE table + expression (or z-score) matrix** (paths in yaml); curated scripts **rerun** upstream matrix prep if needed, then build the **ggplot2** heatmap using **`manuscript_matrix_heatmap_ggplot()`** (or the same structure inline) from this skill — **no** `file.copy` of exploratory PNG as the final figure.
6. **Volcano / ranked plots** — rebuild from **DE CSV** with the same test and thresholds as the paper, using **`manuscript_volcano_de_ggplot()`** / **`manuscript_volcano_limma_fdr_ggplot()`** from this skill as the style baseline; avoid hand-edited point lists unless explicitly documented.

**`figure_manifest.yaml`:** set top-level **`reproduction_lineage_file`** to `config/reproduction_lineage.yaml`. Each figure entry should keep **`source_exploratory.pipeline_entry`** pointing at the **earliest** script in the chain that still applies (often DE `R.01`), **`pipeline_script`** for the step that produced the panel’s scientific inputs, and **`inputs`** listing every file the curated `build_*` reads.

**Publication bundle:** git tag + Zenodo archive of **code + small tables**; large matrices via **GEO / SRA** with checksums referenced in yaml. One top-level command sequence in `README.md` (if allowed), e.g. `snakemake ...` then `Rscript MANUSCRIPT_ROOT/F2/scripts/...`.

## Do not

- Do not move or delete the original exploratory scripts unless the user explicitly asks.  
- Do not silently pick a wrong revision when multiple outputs exist.  
- Do not **`file.copy`** exploratory PNG/PDF into **`<bundle>/figures/`** as the primary curated output: **regenerate** with R (ggplot, upstream `Rscript`, or in-script drawing) so figures stay reproducible; **exports** must follow **§3**; **visual style** on **收录** must match the **source** (see **Preserve source style**), not a stealth full **§2C** restyle.  
- Do not put panels or exports for **one** journal figure inside **another** figure’s bundle.
