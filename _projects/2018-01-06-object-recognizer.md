---
layout: post
title: "E-Commerce Object Recognizer"
date: 2018-01-06
---

E-Commerce Object Recognizer.

A web service which helps people find the name of the products by just taking a photo with their cell phone.

The tools used:
- YOLO - Real Time Object Detection
- TensorFlow For Poets
- Flask and Python Libs
- Web Driver

Methodology:
- The object with highest accuracy is detected, extracted and cropped from the image.
- This cropped image is now given as an input to the classifier that we trained.
- The classifier then predicts the brand of the image.
- We used a pre-trained model from image-net and we retrained the classification layer of our model to predict and classify a few brands.
- We used the Python Flask framework as our web server and HTML for the web app.

Future work:
- Improve training using a bigger dataset and.
- Try real time object detection and predictiction through camera.
- Accurate predictions would further enable automated web crawling and integration of price comparison systems.


