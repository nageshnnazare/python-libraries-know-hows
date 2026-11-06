# 12 - Matplotlib Advanced

Complex layouts, annotations, twin axes, 2-D/statistical plots, styling for
publication, 3-D, and animation. Plus a short note on Seaborn (built on
matplotlib) even though it was outside the core scope, because you will encounter
it.

```python
import matplotlib.pyplot as plt
import numpy as np
rng = np.random.default_rng(0)
```

## 1. Advanced layouts

### subplot_mosaic (named, non-uniform grids)

```python
fig, axd = plt.subplot_mosaic(
    """
    AAB
    CCB
    """,
    figsize=(10, 6),
)
axd["A"].plot([1, 2, 3])       # top-left, spans 2 columns
axd["B"].hist(rng.normal(size=500))   # right column, spans 2 rows
axd["C"].scatter(rng.random(50), rng.random(50))
```

```
   Mosaic string maps letters to grid cells:
   "AAB"     A A B      A occupies top two cells (wide)
   "CCB"     C C B      B occupies the right two cells (tall)
                        C occupies bottom two cells (wide)
```

`subplot_mosaic` is the modern, readable way to build dashboards with spanning
panels. Alternatives: `GridSpec` (finer control) and `fig.add_axes([l,b,w,h])`
for fully manual placement (used for insets).

### GridSpec for fine control

```python
from matplotlib.gridspec import GridSpec
fig = plt.figure(figsize=(10, 8))
gs = GridSpec(3, 3, figure=fig, hspace=0.3, wspace=0.3)
ax_main = fig.add_subplot(gs[:2, :2])     # big main plot (2x2)
ax_top  = fig.add_subplot(gs[2, :2])      # bottom strip
ax_side = fig.add_subplot(gs[:2, 2])      # right strip
```

### Inset axes (zoom / overview)

```python
ax_inset = ax.inset_axes([0.6, 0.6, 0.35, 0.35])   # [x, y, w, h] in axes coords
ax_inset.plot(x, y)
ax.indicate_inset_zoom(ax_inset)                    # draw connector lines
```

## 2. Twin axes (two y-scales)

```python
fig, ax1 = plt.subplots()
ax1.plot(years, revenue, "b-")
ax1.set_ylabel("Revenue ($M)", color="b")

ax2 = ax1.twinx()                 # shares x, independent y on the right
ax2.plot(years, margin, "r--")
ax2.set_ylabel("Margin (%)", color="r")
```

```
   twinx(): two datasets, same x-axis, different y-scales
        Revenue ($M)  ^                          ^  Margin (%)
                      │   ___                     │
                      │  /   \  ___              │
                      └───────────────────────────>
                                 years
   Use sparingly: dual axes can mislead by implying correlation. Consider two
   stacked subplots instead.
```

## 3. Annotations and text

```python
fig, ax = plt.subplots()
ax.plot(x, y)
ax.text(2, 0.5, "note", fontsize=12, color="red")          # text at data coords
ax.annotate("peak",
            xy=(1.57, 1.0),                # point to annotate (data coords)
            xytext=(3, 0.7),               # text location
            arrowprops=dict(arrowstyle="->", color="black"),
            fontsize=11)
ax.axhspan(0.8, 1.0, alpha=0.2, color="green")   # shaded horizontal band
ax.axvspan(1, 2, alpha=0.1, color="red")         # shaded vertical band

# Coordinate systems for text:
# transform=ax.transData    (default) -> data coordinates
# transform=ax.transAxes    -> 0..1 relative to the axes (corner labels)
ax.text(0.02, 0.95, "A", transform=ax.transAxes, fontweight="bold")
```

Annotations turn a plot into an explanation. Use `transAxes` (0-1 range) to place
labels at fixed corners regardless of data scale.

## 4. 2-D and statistical plots

```python
# Heatmap of a matrix / correlation
mat = rng.random((10, 10))
fig, ax = plt.subplots()
im = ax.imshow(mat, cmap="viridis", aspect="auto")
fig.colorbar(im, ax=ax)

# Correlation heatmap with annotations
corr = df.corr(numeric_only=True)
fig, ax = plt.subplots()
im = ax.imshow(corr, cmap="coolwarm", vmin=-1, vmax=1)
ax.set_xticks(range(len(corr))); ax.set_xticklabels(corr.columns, rotation=45)
ax.set_yticks(range(len(corr))); ax.set_yticklabels(corr.columns)
for i in range(len(corr)):
    for j in range(len(corr)):
        ax.text(j, i, f"{corr.iloc[i,j]:.2f}", ha="center", va="center")
fig.colorbar(im)

# 2-D density
ax.hist2d(x, y, bins=30, cmap="Blues")           # gridded density
ax.hexbin(x, y, gridsize=30, cmap="Blues")       # hexagonal binning

# Contours (for f(x,y) surfaces)
X, Y = np.meshgrid(np.linspace(-3, 3, 100), np.linspace(-3, 3, 100))
Z = np.exp(-(X**2 + Y**2))
cs = ax.contourf(X, Y, Z, levels=20, cmap="viridis")
ax.contour(X, Y, Z, levels=10, colors="k", linewidths=0.5)
fig.colorbar(cs)

# Error bars and filled bands
ax.errorbar(x, y, yerr=err, fmt="o", capsize=3)
ax.fill_between(x, y - err, y + err, alpha=0.3)   # confidence band
```

`fill_between` for confidence/uncertainty bands and `imshow`/heatmaps for
matrices are staples of data-science plotting.

## 5. Publication-quality styling

```python
# Set global defaults via rcParams (affects all subsequent plots)
plt.rcParams.update({
    "figure.dpi": 120,
    "font.size": 11,
    "font.family": "sans-serif",
    "axes.spines.top": False,
    "axes.spines.right": False,
    "axes.grid": True,
    "grid.alpha": 0.3,
    "figure.autolayout": True,
})

# Or use a context manager to apply temporarily
with plt.style.context("ggplot"):
    fig, ax = plt.subplots()
    ax.plot(x, y)

# LaTeX-style math in labels (no LaTeX install needed for mathtext)
ax.set_xlabel(r"$\alpha$ (radians)")
ax.set_title(r"$y = \sin(\alpha)$, $\sigma^2 = 1$")
```

```
   Checklist for a clean, honest figure:
   [OK] descriptive title + axis labels with units
   [OK] legend when >1 series; direct labels are even better
   [OK] remove chartjunk (top/right spines, heavy gridlines)
   [OK] colorblind-safe palette (viridis/tab10), not jet
   [OK] axes start at a sensible baseline (bar charts: start y at 0)
   [OK] enough dpi / vector format for the medium
```

## 6. 3-D plotting

```python
fig = plt.figure(figsize=(8, 6))
ax = fig.add_subplot(111, projection="3d")   # request a 3-D axes
ax.plot_surface(X, Y, Z, cmap="viridis", alpha=0.9)
ax.scatter(xs, ys, zs)
ax.set_xlabel("x"); ax.set_ylabel("y"); ax.set_zlabel("z")
ax.view_init(elev=30, azim=45)               # camera angle
```

3-D plots in matplotlib are static and often hard to read; for interactive 3-D,
Plotly (file 15) is usually the better choice.

## 7. Animation

```python
from matplotlib.animation import FuncAnimation

fig, ax = plt.subplots()
line, = ax.plot([], [])
ax.set_xlim(0, 2*np.pi); ax.set_ylim(-1, 1)

def update(frame):
    xs = np.linspace(0, 2*np.pi, 100)
    line.set_data(xs, np.sin(xs + frame / 10))
    return line,

anim = FuncAnimation(fig, update, frames=100, interval=50, blit=True)
anim.save("wave.gif", writer="pillow", fps=20)   # or .mp4 with ffmpeg
```

`blit=True` redraws only changed artists (faster). Saving GIF needs `pillow`; MP4
needs `ffmpeg` installed.

## 8. Interactive backends

```python
# In Jupyter:
# %matplotlib inline      static images (default)
# %matplotlib widget      interactive pan/zoom (needs ipympl)
# In scripts, plt.show() opens an interactive window (backend-dependent).
```

For genuinely interactive, shareable charts (hover tooltips, zoom, linked views),
use Plotly (file 15) rather than fighting matplotlib backends.

## 9. Seaborn (brief, for awareness)

Seaborn sits on top of matplotlib and produces attractive statistical plots with
far less code. It was outside the agreed scope, but you will meet it constantly.

```python
# import seaborn as sns
# sns.scatterplot(data=df, x="age", y="salary", hue="dept", size="bonus")
# sns.histplot(df["salary"], kde=True)
# sns.boxplot(data=df, x="dept", y="salary")
# sns.heatmap(df.corr(numeric_only=True), annot=True, cmap="coolwarm")
# sns.pairplot(df, hue="dept")     # scatter matrix of all numeric pairs
```

Because Seaborn returns matplotlib Axes, you can post-process with the OO API you
already know. If you want, I can add a dedicated Seaborn file; it was not in the
original scope you selected.

## Exercises

1. Build a dashboard with `subplot_mosaic`: a wide time-series panel on top and
   two smaller panels (histogram, scatter) below.
2. Make a twin-axis chart of revenue (left) and margin percent (right); then argue
   in a comment why two stacked subplots might be better.
3. Draw an annotated line plot marking the maximum with an arrow and a shaded band
   around a region of interest.
4. Render an annotated correlation heatmap of a numeric DataFrame with a diverging
   colormap centered at 0.
5. Create and save a short `FuncAnimation` of a moving sine wave as a GIF.

Proceed to `13_scipy.md`.
