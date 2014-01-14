---
layout: post
title:  GigaSpaces-Excel Troubleshooting
categories: SBP
parent: excel-that-scales-solution.html
weight: 600
---

{% summary page|65 %}Debugging using Excel; how to write a hosting application for easy debugging.{% endsummary %}

# Overview

# Writing a Hosting Application for Easy Debugging

{% lampon %} If it's the first time you are running UDF of RTD, try running a simple example that only returns a value, to check that the API works in your environment.

Writing applicative UDF/RTD:

1. Comment the space API calls inside UDF/RTD code and make it return constant values.
2. Create a console or Win-form application that hosts the UDF/RTD DLL and debug it directly to solve the function implementation problems.
