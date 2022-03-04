# KubeCon Europe 2020 – Tilt Demo

The demo has two stages. One is the typical 1–2 minute demo showing Tilt's basic funcionality. Two is a longer demo going deeper into functionality, for those users who wish a more in-depth presentation.

The former is priority but we should—even if briefly—discuss what features to show, and how to demo them, regarding the latter.

## Setup

We'll be running `github.com/windmilleng/enhance/terminator/`, it should be rename "Image Filters" or something like that.

## Basic Idea

1. What the app does
    - Go over the front-end and show the different image filters
1. Explain how troublesome it is to make changes without Tilt
    - `docker build`, `docker push`, `kubectl apply`, etc.
1. Show Tilt's normal workflow by changing one of the filters
1. Show Tilt with live reload & hacks, then change another filter
1. Force an error, show it on dashboard
1. Take a snapshot
1. (Optional) Toggle local ↔ remote cluster

## Issues

- The "hacker look." I'll attach in this PR's comments the output after a few simple tweaks. It's easy to "rebrand" this and make it look non-hackery.