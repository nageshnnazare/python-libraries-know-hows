# 15 - Plotly (Interactive Visualization)

Plotly produces interactive, web-based charts: hover tooltips, zoom/pan, toggling
series, and 3-D you can rotate. Use it for exploration, dashboards, and anything
shared in a browser. For static publication figures, matplotlib is still simpler.

```python
import plotly.express as px          # high-level, one-liners (start here)
import plotly.graph_objects as go    # low-level, full control
import pandas as pd
import numpy as np
```

## 1. matplotlib vs Plotly: when to use which

```
   ┌───────────────────┬──────────────────────────────────────────────┐
   │ matplotlib        │ Plotly                                        │
   ├───────────────────┼──────────────────────────────────────────────┤
   │ static images     │ interactive (hover, zoom, pan, toggle)        │
   │ publication/print │ web, dashboards, notebooks, sharing           │
   │ pixel-level ctrl  │ fast good-looking defaults                    │
   │ PNG/PDF/SVG       │ HTML (self-contained) or embedded             │
   │ small footprint   │ heavier output (JS bundle)                    │
   └───────────────────┴──────────────────────────────────────────────┘
```

## 2. Plotly Express: the fast path

Express creates a complete figure from a tidy DataFrame in one call. It expects
long/tidy data (file 09).

```python
df = px.data.gapminder()      # built-in sample dataset

fig = px.scatter(df.query("year == 2007"),
                 x="gdpPercap", y="lifeExp",
                 size="pop", color="continent",
                 hover_name="country", log_x=True,
                 size_max=60, title="Wealth vs Health (2007)")
fig.show()                    # opens in browser / renders inline in Jupyter
```

```
   px maps DataFrame columns to visual channels:
   x, y         -> position
   color        -> hue (categorical or continuous)
   size         -> marker size
   symbol       -> marker shape
   facet_row / facet_col -> small multiples (one subplot per category)
   animation_frame       -> a play button across a column's values
   hover_name / hover_data -> tooltip contents
```

### Common Express chart types

```python
px.line(df, x="year", y="gdpPercap", color="country")        # trends
px.bar(df, x="continent", y="pop", color="continent")        # categories
px.histogram(df, x="lifeExp", nbins=30, color="continent")   # distribution
px.box(df, x="continent", y="lifeExp")                       # distribution+outliers
px.violin(df, x="continent", y="lifeExp", box=True)          # density
px.scatter_matrix(df, dimensions=["gdpPercap", "lifeExp", "pop"])  # pair plot
px.density_heatmap(df, x="gdpPercap", y="lifeExp")           # 2-D density
px.imshow(corr_matrix, text_auto=True, color_continuous_scale="RdBu")  # heatmap
px.sunburst(df, path=["continent", "country"], values="pop")  # hierarchy
px.treemap(df, path=["continent", "country"], values="pop")
px.choropleth(df, locations="iso_alpha", color="lifeExp")     # maps
```

### Faceting and animation (Express superpowers)

```python
# Small multiples: one panel per continent
px.scatter(df, x="gdpPercap", y="lifeExp", color="continent",
           facet_col="continent", facet_col_wrap=3, log_x=True)

# Animated over time: a play/pause slider
px.scatter(df, x="gdpPercap", y="lifeExp",
           size="pop", color="continent", hover_name="country",
           animation_frame="year", animation_group="country",
           log_x=True, size_max=60, range_y=[25, 90])
```

Faceting and animation in one argument each are why Express is so productive for
EDA; the matplotlib equivalents take dozens of lines.

## 3. Graph Objects: full control

Every Express figure is a `go.Figure` under the hood. Use Graph Objects directly
when you need fine control or to combine trace types.

```python
fig = go.Figure()
fig.add_trace(go.Scatter(x=x, y=y1, mode="lines", name="series 1"))
fig.add_trace(go.Scatter(x=x, y=y2, mode="markers", name="series 2"))
fig.add_trace(go.Bar(x=cats, y=vals, name="bars"))

fig.update_layout(
    title="Custom figure",
    xaxis_title="x", yaxis_title="y",
    template="plotly_white",           # themes: plotly, plotly_dark, ggplot2, ...
    hovermode="x unified",             # single tooltip across traces at an x
    legend=dict(orientation="h", y=1.1),
    width=800, height=500,
)
fig.update_xaxes(showgrid=True, range=[0, 10])
fig.update_yaxes(type="log")
fig.show()
```

```
   A Plotly figure has two parts:
   fig.data    a list of traces   (Scatter, Bar, Heatmap, ... = the data)
   fig.layout  everything else    (title, axes, legend, template, annotations)

   Express fills both automatically; go.Figure lets you build them piece by piece.
   You can even start with px and then .update_layout()/.add_trace() on it.
```

### Scatter modes

```python
go.Scatter(x=x, y=y, mode="lines")           # line only
go.Scatter(x=x, y=y, mode="markers")         # scatter only
go.Scatter(x=x, y=y, mode="lines+markers")   # both
go.Scatter(x=x, y=y, fill="tozeroy")         # area chart
```

## 4. Subplots

```python
from plotly.subplots import make_subplots

fig = make_subplots(rows=2, cols=2,
                    subplot_titles=("A", "B", "C", "D"),
                    specs=[[{"type": "xy"}, {"type": "xy"}],
                           [{"type": "domain"}, {"type": "scene"}]])  # pie, 3d
fig.add_trace(go.Scatter(x=x, y=y), row=1, col=1)
fig.add_trace(go.Bar(x=cats, y=vals), row=1, col=2)
fig.add_trace(go.Pie(labels=labels, values=vals), row=2, col=1)
fig.update_layout(height=700, showlegend=False)
```

## 5. 3-D and specialized plots

```python
# 3-D scatter you can rotate with the mouse
px.scatter_3d(df, x="a", y="b", z="c", color="group")

# 3-D surface
go.Figure(go.Surface(z=Z, x=X, y=Y))

# Financial candlestick
go.Figure(go.Candlestick(x=dates, open=o, high=h, low=l, close=c))

# Parallel coordinates (multivariate)
px.parallel_coordinates(df, color="target",
                        dimensions=["f1", "f2", "f3", "f4"])
```

3-D is where Plotly clearly beats matplotlib: rotation and zoom make 3-D data
actually interpretable.

## 6. Styling and annotations

```python
fig.update_layout(template="plotly_dark")     # dark theme
fig.update_traces(marker=dict(size=10, line=dict(width=1, color="white")))
fig.add_hline(y=50, line_dash="dash", annotation_text="threshold")
fig.add_vline(x=5, line_color="red")
fig.add_annotation(x=3, y=8, text="peak", showarrow=True, arrowhead=2)
fig.add_shape(type="rect", x0=1, x1=3, y0=0, y1=10,
              fillcolor="lightblue", opacity=0.2, line_width=0)

# Continuous and discrete color scales
px.scatter(df, x="a", y="b", color="value",
           color_continuous_scale="Viridis")
px.bar(df, x="cat", y="v", color="group",
       color_discrete_sequence=px.colors.qualitative.Set2)
```

## 7. Exporting and embedding

```python
fig.write_html("chart.html")                 # self-contained interactive file
fig.write_html("chart.html", include_plotlyjs="cdn")   # smaller, needs internet
fig.write_image("chart.png", scale=2)        # static export (needs 'kaleido')
fig.write_image("chart.pdf")
fig.to_json()                                # serialize for web apps

# In Jupyter, fig.show() renders inline. In scripts it opens a browser tab.
```

```
   Output choice:
   write_html   share an interactive chart with anyone (opens in a browser)
   write_image  static PNG/PDF/SVG for docs/slides (pip install kaleido)
   to_json      hand off to a web front-end / Dash app
```

## 8. Dashboards (Dash, brief)

For full interactive apps (dropdowns, callbacks, live data), Plotly's companion
framework Dash turns figures into web apps.

```python
# pip install dash
# from dash import Dash, dcc, html, Input, Output
# app = Dash(__name__)
# app.layout = html.Div([
#     dcc.Dropdown(id="year", options=sorted(df.year.unique()), value=2007),
#     dcc.Graph(id="scatter"),
# ])
# @app.callback(Output("scatter", "figure"), Input("year", "value"))
# def update(year):
#     return px.scatter(df.query("year == @year"), x="gdpPercap", y="lifeExp")
# app.run(debug=True)
```

Dash is the path from a single chart to a shareable interactive dashboard without
writing JavaScript. It was outside the core scope but is the natural next step.

## 9. Practical guidance

```
   [OK] Start with Plotly Express; drop to Graph Objects only when needed.
   [OK] Feed Express tidy/long DataFrames (melt wide data first).
   [OK] Use hover_data to expose extra columns in tooltips for EDA.
   [OK] Use facets/animation for multi-dimensional exploration.
   [WARN] HTML output embeds plotly.js (~3 MB); use include_plotlyjs="cdn"
          to shrink files when viewers have internet.
   [WARN] Huge scatter plots (>~100k points) get sluggish; use
          px.scatter with render_mode="webgl" (Scattergl) or downsample.
```

## Exercises

1. Load `px.data.gapminder()` and recreate the animated bubble chart (gdp vs life
   expectancy over years, bubble size = population, color = continent).
2. Build a faceted histogram of a numeric column split by a categorical column.
3. Start a figure with Express, then use `add_hline`, `add_annotation`, and
   `update_layout(template=...)` to customize it.
4. Make a 2x2 subplot grid mixing a line, a bar, a pie, and a 3-D scatter.
5. Export one interactive chart to standalone HTML and one static chart to PNG.

Proceed to `16_workflow_best_practices.md`.
