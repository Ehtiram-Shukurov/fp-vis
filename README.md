# Dimension

**How does the demographic of users change how they watch movies?**

A scrollytelling data story built for CSCI 5609 — Data Visualization, Spring 2026.  
Live site: [ehtiram-shukurov.github.io/fp-vis](https://ehtiram-shukurov.github.io/fp-vis/)

---

## Team

Dylan Lyon · Genevieve Gray · Ezra Shukurov · Jake O'Shaughnessy · Max LaLonde

---

## About

Dimension is a scroll-driven data story that explores how age, gender, and occupation shape the way people rate movies. It is built on the MovieLens 100K dataset, which contains 100,000 ratings from 943 users across 1,682 movies, collected between 1997 and 1998.

The project guides the reader through a series of curated findings, then hands control over to them to explore the data themselves.

---

## Dataset

**MovieLens 100K** — GroupLens Research, University of Minnesota  
[grouplens.org/datasets/movielens](https://grouplens.org/datasets/movielens/)

The dataset is split across three files:

- `u.data` — individual ratings (user id, movie id, rating, timestamp)
- `u.item` — movie metadata (title, release date, genres)
- `u.user` — user demographics (age, gender, occupation, zip code)

All three files are fetched at runtime and joined in the browser. No backend is needed.

---

## Sections

### 01 · Occupation × Genre
A scrollytelling histogram walking through how different occupations rate different genres. Healthcare workers rate Horror significantly lower than almost any other group. Lawyers rate it higher. The final step unlocks a fully interactive histogram where you can filter by genre, movie, age, gender, and occupation, and compare two distributions side by side.

### 02 · Age × Genre
A multi-line chart showing how average genre ratings shift across age groups. Film-Noir climbs steadily from the youngest viewers to the oldest. Horror goes the other direction. Each scroll step highlights a curated genre comparison, and the final step unlocks all 18 genre toggles for free exploration. Dot size on each point encodes the number of ratings it is based on.

### 03 · How Do You Compare?
A categorical scatterplot where you pick a movie, enter your own demographics and rating, and see exactly where you land relative to everyone else in the dataset. The x-axis shifts between age, gender, and occupation as you scroll. Dot size represents how many people gave that rating.

### 04 · Glyph Exploration
A glyph grid showing every age × occupation combination at once. Each glyph is a radial chart where each spoke represents a genre and spoke length represents the average rating for that group. Hover any glyph for details, click to enlarge.

### 05 · Chord Chart
A chord diagram connecting user demographics to their average rating. Each chord represents a user, and the visualization can be filtered by occupation. Inspired by a chart the class filled out on the first day of the course.

---

## Tech Stack

- **Framework**: SvelteKit with Svelte 5 runes
- **Visualization**: D3.js
- **Deployment**: GitHub Pages
- **Fonts**: Outfit via Google Fonts
- **Styling**: Component-scoped CSS with global CSS variables in `app.css`

---

## Running Locally

```bash
npm install
npm run dev
```

Then open [localhost:5173](http://localhost:5173).

To build for production:

```bash
npm run build
```

---

## References & Acknowledgements

- MovieLens 100K Dataset — F. Maxwell Harper and Joseph A. Konstan. 2015. The MovieLens Datasets. ACM Transactions on Interactive Intelligent Systems.
- Categorical Scatterplot concept: [datylon.com](https://www.datylon.com/blog/types-of-charts-graphs-examples-data-visualization)
- Glyph inspiration: [OECD Better Life Index](https://www.oecd.org/en/data/tools/well-being-data-monitor/better-life-index.html)
- Counting animation: [Medium](https://medium.com/@phornphatch/creating-a-counting-number-animation-from-0-to-with-pure-css-9a1004cf5c91)
- D3.js line chart reference: [D3 Graph Gallery](https://d3-graph-gallery.com/line.html)

Thanks to our peer reviewers, quiz respondents, TA Pan Hao, and Professor Wang.
