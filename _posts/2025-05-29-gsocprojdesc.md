---
layout: post
title: Introduction to Weighted Ensemble Simulations and MDAnalysis Streaming
date: 2025-05-29 00:00:00 +0100
description: Intro to WE and MDAnalysis Streaming # Add post description (optional)
img: gsocheader.png # Add image post (optional)
tags: [GSOC, Molecular dynamics] # add tag
---
As mentioned in my last post, I am thrilled to have been accepted for Google Summer of Code 2025 with MDAnalysis and WESTPA. In this post, I am going to dive deeper into the details of my project, which focuses on leveraging MDAnalysis trajectory streaming to perform on-the-fly analysis of multiple trajectories generated from Weighted Ensemble simulations.

# Introduction to Weighted Ensemble Simulations
Weighted Ensemble simulations are a powerful technique in molecular dynamics that allow for efficient exploration of rare events and free energy landscapes. Weighted Ensemble methods are particularly appealing as they permit the calculation of kinetics and thermodynamics of complex processes without requiring the application of biasing potentials. In the Weighted Ensemble method, many short simulations are run in parallel. The simulations are run in iterations, where after each iteration, progress coordinates are calculated, and the simulations are duplicated or culled based on their progress. WESTPA (Weighted Ensemble Simulation Toolkit for Parallel Analysis) is state-of-the-art software for performing Weighted Ensemble simulations. It provides a framework for running and managing these simulations efficiently.

# The Challenge of Analysis Overhead
One of the significant challenges in performing Weighted Ensemble simulations is the analysis overhead. After each iteration, the trajectory data needs to be loaded and the progress coordinates need to be calculated,.