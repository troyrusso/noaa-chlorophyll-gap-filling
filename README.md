# Mind the Gap: Gap-Filling Satellite Chlorophyll-a with Deep Learning

**NOAA Fisheries · Varanasi Summer Fellow (SAFS Varanasi Internship), Summer 2026**
Mentor: Eli Holmes, Ph.D. (NOAA Fisheries and UW School of Aquatic and Fishery Sciences)

**A full summary of my contributions is in my [capstone notebook](https://github.com/SAFS-Varanasi-Internship/mindthegap/blob/troy-branch/final_notebooks/Mind_the_Gap_Capstone.ipynb).** The code is in the [project repository](https://github.com/SAFS-Varanasi-Internship/mindthegap). This page is a short overview.

---

## The problem

Satellites such as PACE and Copernicus GlobColour measure chlorophyll-a, which tracks phytoplankton at the base of the marine food web. Their daily fields are full of holes: clouds block the sensor, and polar-orbiting instruments see only narrow swaths, so a single day can be mostly empty.

## The approach

We train a U-Net to fill the gaps, supervised by the data itself. The model sees observed pixels with some of them hidden behind synthetic clouds, and learns to put the hidden values back. Because the truth is known for the hidden pixels, the fill can be scored without outside labels. At inference the same model fills the real gaps. The project builds on work from GeoHackWeek 2024 and OceanHackWeek 2025.

## What I did

* **Traced and fixed a tiling artifact.** Models trained on small tiles printed a seam texture into their fills. I traced it to synthetic clouds being cut at tile edges during training, and ruled out undertraining, the input channels, and the transposed-convolution checkerboard. The artifact disappears once tiles are about ten times the cloud size (about 128 px here), and the same threshold held on two datasets six times apart in resolution.
* **Scaled training from one ocean basin to the globe.** Streamed the data one tile at a time so a basin trains in about 0.3 GB of memory regardless of region size. Moved training to SkyPilot managed jobs on AWS, then built a region-by-region global run: eight longitude strips, each cached locally and trained in its own subprocess, which avoids both host-memory limits and the one-hour expiry of NASA's S3 credentials. The run completed in 4 hours 20 minutes on a single A10G GPU and produced the project's first global model.
* **Tested the evaluation, not just the model.** Showed that independent synthetic clouds let the model copy neighboring days: the neighbor-day inputs were worth about 21% of the score, falling to about 4% once clouds persisted across days. Also showed that scoring per frame versus per pixel can flip the comparison with persistence (yesterday's value). The model adds the most on large gaps that persistence cannot fill; on easy open water, persistence is hard to beat.
* **Ran the pipeline on a second sensor.** Applied the same pipeline to Copernicus GlobColour at 4 km, and built a joint model that fills several phytoplankton groups at once (it works, but needs a denser data record than the early years we loaded to test it properly).

Two changes that made the global fills work, training on the full field with plain MSE and relabeling the gap flag at inference, came from Eli's streamlined pipeline. I worked with UW eScience data scientists on the cloud setup.

## Figures

![Tiling artifact versus training tile size](https://raw.githubusercontent.com/SAFS-Varanasi-Internship/mindthegap/troy-branch/artifactvstilesize.png)

**Seam artifact versus training tile size, Indian Ocean.** The seam texture is visible at 88 and 112 px, and gone at 128 px and above.

![Global gap-fill example](https://raw.githubusercontent.com/SAFS-Varanasi-Internship/mindthegap/troy-branch/final_notebooks/global_gapfill_example.png)

**One strip from the region-by-region global model, September 2024 (west Pacific and Indonesia).** Left: observed input, mostly swath gaps. Right: the gap-filled field. The model was trained one strip at a time and never saw the whole globe at once.

## Status

I am continuing on the project as a UW research assistant through June 2027: writing up and presenting the summer results, then evaluating the approach against alternative gap-filling methods.

## Tools

Python, TensorFlow/Keras (U-Net), xarray, zarr, xbatcher, SkyPilot, AWS, pixi, Hugging Face buckets, Jupyter Book
