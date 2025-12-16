---
title: "Learning Reproducibility Without a Textbook"
date: 2025-12-15
tags: ["reproducibility", "R", "workflow", "data", "geoinformatics"]
---

Last week I presented the reproducibility infrastructure we built for a [COVID-19 mortality study](https://doi.org/10.3390/systems13110971) to a group of master's students. About 14 people showed up, smaller than we'd hoped, but shit happens.

One student asked for the GitHub repo link. Another asked how to learn R. Simple questions. But watching them engage with the work made something clear: **reproducibility isn't about technical perfection. It's about being useful to the next person who touches your work.**

There's no course that teaches this. You just figure it out by thinking about who's going to use what you built and what they'll need when they try.

For our project, I had two people in mind the entire time: the auditor who wants to break the code, and the interested reader who wants to understand it. Every decision you're about to read? Built for one of those two people.

> But before I'm rambling like an LLM and eating your attention span as breakfast...

---

## What "reproducibility" actually means (and why it matters)

The [National Academies of Sciences](https://www.ncbi.nlm.nih.gov/books/NBK547531/) defines it like this:

> **Reproducibility** is obtaining consistent results using the same input data; computational steps, methods, and code; and conditions of analysis.

In other words: someone else takes your data and your code, runs it, and gets the same numbers you did. Bitwise identical, ideally. Or at least within an acceptable range of variation.

This is different from **replicability**, which is:

> Obtaining consistent results across studies aimed at answering the same scientific question, each of which has obtained its own data.

**Translation:**
- **Reproducibility** = same data, same code, same result
- **Replicability** = new data, similar methods, consistent conclusion

For this post, I'm focusing on **reproducibility**... well, the computational kind. Can someone else run your pipeline and verify your claims? That's the bar.

## Reproducibility in practice (the usual advice)

Standard reproducibility advice looks like this:

- Use version control (Git)
- Document your code (comments, README)
- Pin your dependencies (`requirements.txt` in Python, `renv` in R)
- Write a clear methods section
- Share your data (if legally/ethically possible)
- ???
- Profit

It's like saying "write good code" or "communicate clearly" - true, but how? Because it doesn't tell you *why* those things matter, or *when* they actually help, or *who* they're for. It's prescriptive rules without the underlying logic.

Here's what actually works: **Think about your stakeholders.** Big surprise, I know.

## The two-stakeholder framework

### The Auditor

This person wants to *break* your analysis. They're looking for:
- "Which version of the data did you use?"
- "What happens if Kosovo isn't in your ISO3 mapping?"
- "Can you prove this result didn't come from a random seed you forgot to document?"

**Design response:**
- Data provenance (snapshot dates, licensing)
- Robust error handling (pipeline doesn't crash on weird edge cases)
- Transparency reports (document what failed and why)

### The Interested Reader

This person wants to *understand* your work. They're asking:
- "Can I actually run this on my machine?"
- "Where's the part that does X?"
- "Do I need to re-run everything just to regenerate one figure?"

**Design response:**
- Modular code (each script does one thing)
- Clear structure (`data_raw/`, `data_manipulated/`, `outputs/`)
- Sensible defaults (relative paths, documented dependencies)

## Case study: COVID-19 economy–mortality nexus

We analyzed how **economic development** (GDP), **health systems** (UHC coverage), **demographics** (age), and **vaccination** relate to **excess mortality** across 79 countries. Six data sources. Inconsistent formats. Licensing chaos. The stakeholder lens kept it from becoming a dumpster fire.

### 1) Data provenance (for the auditor)

**The problem:**  
Data sources update at different frequencies. Some frequently, some annually. Values get revised, methodologies change. If you just say "we used World Bank data," your analysis breaks in six months when someone tries to replicate it. They'll pull a newer version with different numbers.

**The solution:**  
Pin snapshot dates in `DATA_SOURCES.md`:
- Median age: March 20, 2025
- Population ages 65+: September 29, 2025
- GDP per capita: December 9, 2024
- UHC service coverage: December 9, 2024
- Excess mortality: April 6, 2025
- Vaccination doses: March 21, 2025

**Why it matters:**  
The auditor asks: "Which version?" You answer: "This exact one, here's the date, here's the license, here's the raw file." No ambiguity. No "well, we got it from somewhere around Q2 2024..."

Also: every dataset has its own `licenses/` subfolder with the actual CC BY 4.0 / WHO terms. The auditor wants to know if they can legally fork this for their thesis. Now they can check without guessing.

### 2) ISO3 standardization (for the auditor again)

**The problem:**  
Six datasets, six different ways of saying "Kosovo." First iteration? Lost 12 countries because `countrycode()` choked on edge cases.

**The solution:**  
Manual overrides in code:
```r
map_iso3 <- function(x) {
  countrycode::countrycode(
    x, origin = "country.name", destination = "iso3c",
    custom_match = c(
      "Kosovo" = "XKX",
      "Micronesia (country)" = "FSM",
      "Virgin Islands" = "VIR",
      "Saint Martin" = "MAF"
    )
  )
}
```

**Why it matters:**  
You *cannot* just run `countrycode()` and pray. The auditor would've caught those 12 missing countries immediately. Now? Zero lost. Edge cases are documented in code, not in excuses.

### 3) Error handling (still for the auditor)

**The problem:**  
We used GAM smoothing to estimate excess mortality at a standardized date (May 5, 2023 - WHO's end of the Public Health Emergency). Some countries have weird time series. Estonia breaks? The whole pipeline crashes? Not acceptable.

**The solution:**  
Wrap the statistical processing in `try()` blocks:
```r
fit <- try(mgcv::gam(y ~ s(x, bs = "cs", k = k_val), data = df), silent = TRUE)

if (inherits(fit, "try-error")) {
  return(tibble(value = NA, method = "gam_failed"))
}
```

At the end, export a **processing summary CSV**: how many countries succeeded with GAM, how many failed, *why* they failed (`gam_failed`, `insufficient_data`, `outside_range`).

**Why it matters:**  
The auditor tests your code with edge cases. The pipeline doesn't crash. It logs failures transparently. That's the difference between "robust code" and "code that only works on your machine with your exact data."

We also quantified uncertainty in the GAM estimates. Relative confidence interval widths ranged from <1% to ~11%, with Armenia showing the highest at 10.74% (see Figure A2 in the paper). High confidence across the board.

### 4) Modular pipeline (for the interested reader)

**The problem:**  
If everything lives in one 2000-line script, the interested reader has to reverse-engineer *everything* just to find the part they care about.

**The solution:**  
Numbered scripts, each doing **one thing**:
```
00_library_loader.R       # Package management
01_loader.R               # Raw data loading, ISO standardization
02_clean.R                # GAM mortality estimation
03_analysis.R             # Correlation analysis
04a_plots_correlations.R  # Figure 3
04b_plots_maps.R          # Figure 2
04c_tables.R              # Tables 1-4
```

Step 2 breaks? Fix Step 2. You don't re-run Step 1 like a psychopath.

**Why it matters:**  
Someone forks your repo for their thesis. They want to see *just* the data loading logic. Boom, `01_loader.R`. They want to regenerate *just* Figure 3. Boom, `04a`. They don't need to wade through 2000 lines and guess which part does what.

And everything uses `here::here()` for paths, so it works on my Windows machine, Christian's Mac, some master student's Linux VM. No `setwd("/Users/max/Documents/...")` nonsense.

### 5) The payoff

**The auditor** can verify our claims:
- Data provenance (snapshot dates, licenses)
- Edge case handling (ISO3, GAM failures)
- Transparent processing (summary CSVs)

**The interested reader** can understand our work:
- Modular structure (each script = one job)
- Clear separation (`data_raw/`, `data_manipulated/`, `outputs/`)
- Works on their machine (relative paths)

And the whole thing? Clone it, source the scripts in order, runs in about 2 minutes. Six data sources, 79 countries, GAM estimation, correlation analysis, all outputs.

**That's reproducibility.**

---

## The meta-point: it's all stakeholder thinking

This pattern shows up everywhere (at least in my life):

- **Bass practice** isn't "play scales for X hours" - it's understanding what you're trying to build and designing practice around that
- **Fitness** isn't "lift weights" - it's compound interest thinking applied to systematic load progression
- **Reproducibility** isn't "follow best practices" - it's stakeholder empathy applied to research infrastructure

**The principle is the same:**  
Before you write code, ask:
1. Who will use this?
2. What will they need?
3. What will break for them?

The tools follow from the answers.

## Practical takeaway

Reproducibility isn't about perfection. It's about **anticipating failure modes**.

- The auditor assumes you made mistakes. Document everything.
- The interested reader assumes you know more than they do. Make it obvious.

You don't need a textbook. You need to think about who's going to touch your work and what they're going to need when they do.

**To students and early-career people:** You've had enough Udacity courses. Enough YT tutorials. Enough "best practices" Linkedin posts. Stop consuming tips and start building projects that make you uncomfortable. That's where you actually learn stuff.

And if you want to see of what I rambled about in action: [here's the GitHub repo](https://github.com/simsynser/economy-mortality-nexus).

**That's reproducibility. Not rules, but empathy.**

---

## References

National Academies of Sciences, Engineering, and Medicine. (2019). *Reproducibility and Replicability in Science*. Washington, DC: National Academies Press. https://www.ncbi.nlm.nih.gov/books/NBK547531/

Neuwirth, C., & Elixhauser, M. (2025). Opposing Effects of National Economic Performance on COVID-19 Mortality. *Systems*, 13(11), 971. https://doi.org/10.3390/systems13110971