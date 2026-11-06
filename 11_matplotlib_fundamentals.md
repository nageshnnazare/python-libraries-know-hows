# 11 - Matplotlib Fundamentals

Matplotlib is the foundational plotting library. It is verbose but total-control.
Learn the object-oriented interface (Figure/Axes); it is the correct mental model
and what every advanced example uses.

```python
import matplotlib.pyplot as plt
import numpy as np
```

## 1. Anatomy of a figure

```
   ┌──────────────────── Figure ─────────────────────┐
   │   (the whole canvas / window)                    │
   │   ┌──────────────── Axes ─────────────────────┐  │
   │   │  Title                                     │  │
   │   │  ^ y-axis label                            │  │
   │   │  │      ● ●                                │  │
   │   │  │    ●     ●   <- data (Line2D, etc.)     │  │
   │   │  │  ●         ●                            │  │
   │   │  └───────────────────> x-axis label       │  │
   │   │     ticks & ticklabels                     │  │
   │   └────────────────────────────────────────────┘  │
   └───────────────────────────────────────────────────┘

   Figure = the page.   Axes = one plot on the page (NOT "axis").
   Axis  = a single x or y number line within an Axes.
```

Key vocabulary confusion: an **Axes** is a whole plot (with its own x and y).
An **Axis** is one number line. A Figure can hold many Axes (subplots).

## 2. Two interfaces: pyplot vs object-oriented

```python
# Implicit pyplot (stateful; fine for quick throwaway plots)
plt.plot([1, 2, 3], [1, 4, 9])
plt.title("quick")
plt.show()

# Object-oriented (explicit; USE THIS for anything real)
fig, ax = plt.subplots()          # create Figure and one Axes
ax.plot([1, 2, 3], [1, 4, 9])
ax.set_title("proper")
plt.show()
```

```
   pyplot style:   plt.plot(); plt.title(); plt.xlabel()
                   -> acts on the "current" axes (implicit global state)
   OO style:       fig, ax = plt.subplots(); ax.plot(); ax.set_title()
                   -> explicit; required for subplots, reusable, unambiguous
```

Always prefer the OO interface. The rest of this tutorial uses `fig, ax`.

## 3. Your first proper plot

```python
x = np.linspace(0, 2 * np.pi, 100)
y = np.sin(x)

fig, ax = plt.subplots(figsize=(8, 4))   # width, height in inches
ax.plot(x, y, label="sin(x)")
ax.set_xlabel("x")
ax.set_ylabel("sin(x)")
ax.set_title("Sine wave")
ax.legend()
ax.grid(True)
fig.tight_layout()                        # prevent label clipping
fig.savefig("sine.png", dpi=150, bbox_inches="tight")
plt.show()
```

```
   Standard build order:
   1. fig, ax = plt.subplots(...)     create canvas + axes
   2. ax.plot / scatter / bar ...     draw data (repeat for multiple series)
   3. ax.set_xlabel / set_title ...   label and annotate
   4. ax.legend()                     add legend (needs label= on plots)
   5. fig.tight_layout()              fix spacing
   6. fig.savefig(...) / plt.show()   output
```

## 4. Line plots and styling

```python
fig, ax = plt.subplots()
ax.plot(x, np.sin(x), color="blue", linewidth=2, linestyle="-", label="sin")
ax.plot(x, np.cos(x), color="red",  linewidth=1, linestyle="--", label="cos")
ax.plot(x, np.sin(2*x), "g:", marker="o", markersize=3, label="sin2x")  # shorthand
ax.legend()
```

```
   Format string shorthand "g:o" = green, dotted line, circle markers.
   color:      "b" "g" "r" "c" "m" "y" "k" "w"  or "#1f77b4" or "tab:blue"
   linestyle:  "-" solid  "--" dashed  ":" dotted  "-." dashdot
   marker:     "o" "." "," "x" "+" "s" square "^" tri "D" diamond "*" star
```

Common line arguments: `color`, `linewidth`/`lw`, `linestyle`/`ls`, `marker`,
`markersize`/`ms`, `alpha` (transparency 0-1), `label`, `zorder` (draw order).

## 5. The core plot types

### Scatter (relationships between two variables)

```python
rng = np.random.default_rng(0)
x = rng.normal(size=200); y = 2 * x + rng.normal(size=200)
fig, ax = plt.subplots()
sc = ax.scatter(x, y, c=y, s=30, cmap="viridis", alpha=0.7, edgecolor="k")
fig.colorbar(sc, ax=ax, label="y value")   # color legend for c=
```

`c=` maps a variable to color, `s=` maps a variable to marker size; together they
encode up to 4 dimensions on a 2-D plot.

### Bar charts (comparing categories)

```python
cats = ["A", "B", "C", "D"]
vals = [23, 45, 12, 38]
fig, ax = plt.subplots()
ax.bar(cats, vals, color="steelblue")        # vertical
# ax.barh(cats, vals)                         # horizontal (better for long labels)

# Grouped bars: offset x positions
x = np.arange(len(cats))
w = 0.35
ax.bar(x - w/2, group1, w, label="2023")
ax.bar(x + w/2, group2, w, label="2024")
ax.set_xticks(x); ax.set_xticklabels(cats); ax.legend()
```

### Histograms (distribution of one variable)

```python
data = rng.normal(100, 15, 1000)
fig, ax = plt.subplots()
ax.hist(data, bins=30, color="skyblue", edgecolor="black", alpha=0.7)
# bins="auto" lets matplotlib choose; density=True normalizes to a PDF
ax.set_xlabel("value"); ax.set_ylabel("count")
```

```
   Choose the plot for the question:
   - trend over an ordered axis (time)       -> line
   - relationship between two numeric vars   -> scatter
   - compare a value across categories       -> bar
   - distribution of one numeric variable    -> histogram
   - distribution + outliers across groups   -> boxplot / violin
   - proportions of a whole (few slices)     -> pie (use sparingly)
```

### Box plots (distribution summary and outliers)

```python
groups = [rng.normal(0, s, 100) for s in (1, 2, 3)]
fig, ax = plt.subplots()
ax.boxplot(groups, tick_labels=["s=1", "s=2", "s=3"])
# box = IQR (Q1..Q3), line = median, whiskers ~1.5*IQR, points = outliers
```

### Pie (use rarely; bars are usually clearer)

```python
fig, ax = plt.subplots()
ax.pie([30, 25, 20, 25], labels=["A","B","C","D"], autopct="%1.1f%%",
       startangle=90)
ax.axis("equal")     # keep it circular
```

## 6. Customizing axes

```python
fig, ax = plt.subplots()
ax.plot(x, y)
ax.set_xlim(0, 10)                    # axis limits
ax.set_ylim(-1.5, 1.5)
ax.set_xticks([0, 5, 10])             # tick positions
ax.set_xticklabels(["start", "mid", "end"])
ax.tick_params(axis="x", rotation=45) # rotate tick labels
ax.set_xscale("log")                  # log scale (also "symlog")
ax.axhline(0, color="gray", ls="--")  # horizontal reference line
ax.axvline(5, color="red")            # vertical reference line
ax.spines["top"].set_visible(False)   # remove chart borders (cleaner look)
ax.spines["right"].set_visible(False)

# Set many properties at once
ax.set(xlabel="x", ylabel="y", title="all at once", xlim=(0, 10))
```

## 7. Multiple subplots

```python
# Grid of axes: subplots returns a Figure and an array of Axes
fig, axes = plt.subplots(2, 2, figsize=(10, 8))
axes[0, 0].plot(x, np.sin(x));  axes[0, 0].set_title("sin")
axes[0, 1].plot(x, np.cos(x));  axes[0, 1].set_title("cos")
axes[1, 0].hist(data)
axes[1, 1].scatter(x[:100], y[:100])
fig.suptitle("Dashboard", fontsize=14)     # figure-level title
fig.tight_layout()

# Shared axes for comparison
fig, axes = plt.subplots(2, 1, sharex=True, figsize=(8, 6))

# Iterate flattened axes
for ax, col in zip(axes.flat, columns):
    ax.plot(df[col])
```

```
   plt.subplots(2, 2) -> axes is a 2x2 array:
   axes[0,0] axes[0,1]
   axes[1,0] axes[1,1]
   axes.flat iterates them in row-major order for looping.
```

For complex, non-uniform layouts use `fig.subplot_mosaic` (covered in file 12).

## 8. Colors, colormaps, and the style system

```python
# Named styles change the whole look
plt.style.use("seaborn-v0_8-darkgrid")     # try "ggplot", "bmh", "fivethirtyeight"
print(plt.style.available)                  # list options

# Colormaps for continuous data
# sequential: "viridis" "plasma" "Blues"   (ordered low->high)
# diverging:  "coolwarm" "RdBu"            (centered, e.g. correlations)
# qualitative:"tab10" "Set2"               (distinct categories)
```

```
   Colormap choice matters for honesty and accessibility:
   sequential  -> magnitude data (0 to max)      use "viridis" (perceptually uniform)
   diverging   -> data around a midpoint (+/-)   use "coolwarm"/"RdBu"
   qualitative -> unordered categories           use "tab10"
   AVOID "jet"/"rainbow": misleading, not colorblind-friendly.
```

## 9. Saving figures

```python
fig.savefig("plot.png", dpi=300, bbox_inches="tight")   # raster, high-res
fig.savefig("plot.pdf")                                  # vector (scales cleanly)
fig.savefig("plot.svg")                                  # vector for web/editing
```

Use PNG at dpi>=200 for slides/docs, and PDF/SVG (vector) for publications where
zooming must stay sharp. `bbox_inches="tight"` trims whitespace and prevents label
clipping.

## 10. Plotting directly from pandas

Pandas wraps matplotlib; convenient for quick exploration.

```python
df["salary"].plot(kind="hist", bins=20)
df.plot(x="age", y="salary", kind="scatter")
df.groupby("dept")["salary"].mean().plot(kind="bar")
df.plot(kind="line", subplots=True, figsize=(10, 8))

# Pass an existing Axes to combine with manual matplotlib:
fig, ax = plt.subplots()
df.plot(x="age", y="salary", kind="scatter", ax=ax)
ax.set_title("from pandas + custom title")
```

`df.plot()` returns the Axes, so you can keep customizing with the OO API. This is
the fastest path for EDA; switch to explicit matplotlib for polished output.

## Exercises

1. Plot sin and cos on one Axes with a legend, grid, labeled axes, and a title;
   save as PNG at 200 dpi.
2. Make a 2x2 subplot grid showing a line, scatter, histogram, and bar chart of
   your own data.
3. Create a grouped bar chart comparing two years across four categories.
4. Build a scatter plot encoding a third variable as color and a fourth as size,
   with a colorbar.
5. Recreate a pandas `df.plot` figure, then use the returned Axes to add a
   horizontal reference line and remove the top/right spines.

Proceed to `12_matplotlib_advanced.md`.
