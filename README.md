[![Contributors][contributors-shield]][contributors-url]
[![Forks][forks-shield]][forks-url]
[![Stargazers][stars-shield]][stars-url]
[![Issues][issues-shield]][issues-url]
[![MIT License][license-shield]][license-url]

<!-- PROJECT LOGO -->
<br />
<div align="center">
  <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj">
    <img src="img/test.jpg" alt="Logo" width="240" height="320">
  </a>

<h3 align="center">2022 UTSW Machine Learning Group Project</h3>

  <p align="center">
   Project 8: Alzheimer's Disease, predicting delta MMSE 
    <br />
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj"><strong>Explore the docs »</strong></a>
    <br />
    <br />
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj">View Demo</a>
    ·
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/issues">Report Bug</a>
    ·
    <a href="https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/issues">Request Feature</a>
  </p>
</div>

<!-- TABLE OF CONTENTS -->
<details>
  <summary>Table of Contents</summary>
  <ol>
    <li>
      <a href="#about-the-project">About The Project</a>
    </li>
    <li>
      <a href="#getting-started">Getting Started</a>
      <ul>
        <li><a href="#preprocessing">Preprocessing</a></li>
        <li><a href="#exploratory-plotting">Exploratory Plotting</a></li>
        <li><a href="#feature-selection">Feature Selection</a></li>
      </ul>
    </li>
    <li>
        <a href="#modeling">Modeling</a>
        <ul>
            <li><a href="#model-selection">Model Selection</a></li>
            <li><a href="#model-training">Model Training</a></li>
            <li><a href="#model-evaluation">Model Evaluation</a></li>
        </ul>
    </li>
    <li><a href="#conclusions">Conclusions</a></li>
    <li><a href="#project-contributions">Project Contributions</a></li>
    <li><a href="#acknowledgments">Acknowledgments</a></li>
  </ol>
</details>


<!-- ABOUT THE PROJECT -->
## About The Project

<figure>
  <img src="./img/projdetails.png", width = "640">
  <figcaption><b>Fig 1.</b> Project Details.</figcaption>
</figure>

<br></br>

What is the MMSE?

<figure>
  <img src="./img/mmseOverview.png", width = "640">
  <figcaption><b>Link: </b> https://oxfordmedicaleducation.com/geriatrics/mini-mental-state-examination-mmse/ </figcaption>
</figure>

<br></br>

Sample MMSE:
<figure>
  <img src="./img/mmse.png", width = "640">
  <figcaption><b>Link: </b> https://cgatoolkit.ca/Uploads/ContentDocuments/MMSE.pdf </figcaption>
</figure>

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- GETTING STARTED -->
## Getting Started

We did some stuff in this project such as:

* This
* That
* Also this
* etc.

### Preprocessing

```{r}
# Read in file
df <- read_xlsx('./data/AD.training.xlsx') %>% 
transform(
  PTID = factor(PTID),
  DX = factor(DX.bl),
  PTGENDER = factor(PTGENDER),
  APOE4 = factor(APOE4)
)

df %>% head()
```

<figure>
  <img src="./img/dfHead.png", width = "1080">
  <figcaption><b>Fig 2.</b> Head of Dataset.</figcaption>
</figure>

<p align="right">(<a href="#top">back to top</a>)</p>

It is important to realize here that our relevant feature space is _extremely_ limited.  Of the 10 original columns in the dataset, 1 (`MMSE.Change`) is the response variable, and 3 (`RID`, `PTID`, `EXAMDATE`) have minimal-no predictive information.  This leaves only 6 columns for training on a dataset with 384 entries. In short, this is very little data for a very complicated problem and this will likely result in relatively high variance in the regressive models.

### Exploratory Plotting

```{r, fig.width= 18, fig.height=18, message = FALSE, warning = FALSE}
df %>% 
  mutate(asinsqrtMMSE = asin(sqrt(minmax(MMSE)))  ) %>%
  select(AGE, PTEDUCAT, MMSE, asinsqrtMMSE, everything(), MMSE.Change, -RID, -PTID, -EXAMDATE) %>% # removing factors with too many levels or irrelevant features
  ggpairs(aes(fill = DX, alpha = 0.7), progress = FALSE)

```

<figure>
  <img src="./img/pairs_colorDX.png", width = "1080">
  <figcaption><b>Fig 3.</b> Exploratory Pair Plot.</figcaption>
</figure>

```{r}
# template: https://www.datanovia.com/en/blog/elegant-visualization-of-density-distribution-in-r-using-ridgeline/
df %>%
  ggplot(aes(x = `MMSE.Change`, y = DX, group = DX, fill = factor(stat(quantile)))) + 
  stat_density_ridges(
    geom = "density_ridges_gradient",
    calc_ecdf = TRUE,
    quantiles = 4,
    quantile_lines = TRUE, 
    jittered_points = TRUE,
    scale = 0.9,
    position = position_points_jitter(width = 0.05, height = 0.1),
    point_size = 1, point_alpha = 0.5, alpha = 0.7) + 
  theme_minimal() +
  scale_fill_brewer(name = 'Quartiles')
```

<figure>
  <img src="./img/mmseBasevsChangeRidgeLine.png", width = "720">
  <figcaption><b>Fig 4.</b> MMSE Ridgeline Plot.</figcaption>
</figure>

Due to the basal distributions being quite similar in both shape and range for the majority of values, it will likely be quite difficult to get a predictor with low variance.

```{r}
df %>%
  select(PTID, `MMSE.Change`, MMSE, DX) %>%
  mutate(MMSE4mo = MMSE + `MMSE.Change`) %>%
  select(-`MMSE.Change`) %>%
  pivot_longer(cols = c(MMSE4mo, MMSE),  names_to = 'time', values_to = 'MMSE') %>%
  ggplot(aes(x = MMSE, y = time, group = PTID, color = DX)) + 
    geom_jitter(width = 0.2, height = 0.075) +
    geom_line() +
    geom_violin(aes(color = NULL, fill = time, group = time), alpha = 0.15, draw_quantiles = c(0.25, 0.5, 0.75)) + 
  theme_minimal() +
  facet_wrap(~DX)
```

<figure>
  <img src="./img/MMSE_densityLinePairs_byDiag.png", width = "720">
  <figcaption><b>Fig 5.</b> Change in MMSE by DX Ridgeline Plot.</figcaption>
</figure>

In some groups (CN/EMCI), the line/points clearly indicate subject variability. This variability seems to increase in LMCI and AD.  Note that because the MMSE score is capped at 30, we may have some censoring which could cause the measure of center to shift to the left.

<p align="right">(<a href="#top">back to top</a>)</p>

### Feature Selection

`RID`, `PTID`, `EXAMDATE` have minimal-no predictive information and we will drop these values for all models.

In order to see if our data can be condensed, we tried PCA and FAMD for dimensionality reduction.

(spoilers, neither helps too much)

* PCA
* FAMD

<p align="right">(<a href="#top">back to top</a>)</p>

## Modeling

* GLM
* RF
* Neural Net?

### GLM

#### Setup
```{r}
gauss <- glm(`MMSE.Change` ~ ., data = df_clean, family = gaussian())
summary(gauss)
plot(gauss)
```

#### Evaluation


<p align="right">(<a href="#top">back to top</a>)</p>

### RF

#### Setup
```{r}
rf <- randomForest(`MMSE.Change` ~ ., data = df_clean, ntrees = 500, importance = TRUE, type = 'regression')
y_pred <- predict(rf)

rf
plot(rf)
importance(rf)
varImpPlot(rf)
```

#### Evaluation


<p align="right">(<a href="#top">back to top</a>)</p>

### Neural Net (?)

#### Setup


#### Evaluation




## Discussion

we found out 
* This thing
* And that thing

## Conclusions

* A short sentence summary of our work
* A brief aside on Limitations
* Future directions may include



<p align="right">(<a href="#top">back to top</a>)</p>

<!-- Project Contributions -->
## Project Contributions

* Austin Marckx - Austin.Marckx@UTSouthwestern.edu
  * I did this thing
  * And that thing

* Noah Chang - WooYong.Chang@UTSouthwestern.edu
  * I did this other thing
  * And that other thing

Project Link: [https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj](https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj)

<p align="right">(<a href="#top">back to top</a>)</p>

<!-- ACKNOWLEDGMENTS -->
## Acknowledgments

Works cited go here...

- MMSE Overview: https://oxfordmedicaleducation.com/geriatrics/mini-mental-state-examination-mmse/
- MMSE Sample Exam: https://cgatoolkit.ca/Uploads/ContentDocuments/MMSE.pdf



<p align="right">(<a href="#top">back to top</a>)</p>


<!-- MARKDOWN LINKS & IMAGES -->
<!-- https://www.markdownguide.org/basic-syntax/#reference-style-links -->
[contributors-shield]: https://img.shields.io/github/contributors/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[contributors-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/graphs/contributors
[forks-shield]: https://img.shields.io/github/forks/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[forks-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/network/members
[stars-shield]: https://img.shields.io/github/stars/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[stars-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/stargazers
[issues-shield]: https://img.shields.io/github/issues/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[issues-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/issues
[license-shield]: https://img.shields.io/github/license/austinmarckx/2022UTSWMachineLearningFinalProj.svg?style=for-the-badge
[license-url]: https://github.com/austinmarckx/2022UTSWMachineLearningFinalProj/blob/master/LICENSE.txts