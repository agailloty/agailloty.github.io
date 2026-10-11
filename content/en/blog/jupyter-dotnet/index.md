---
title: "Coding C# with Jupyter Notebook"
description: "Code C# interactively with Jupyter notebooks"
date: 2024-04-15T18:05:00Z
draft: false
type: "blog"
author: "Axel-Cleris Gailloty"
tags: [dotnet, Csharp, docker]
---

Code C# interactively with Jupyter notebooks.

Being able to quickly experiment with different syntaxes and programming concepts is a fast way to get comfortable with a language and a huge time saver. As a programming enthusiast, I often find myself exploring new languages and testing different approaches to solve problems. Recently, I needed a simple and fast way to test C# and F# syntaxes without having to create a full .NET project every time. That's where Jupyter Lab and Docker entered the picture.

Having used Jupyter notebooks extensively in the Python and R data science world, I realized that integrating languages like C# and F# into that same environment would be incredibly powerful. So I set out to create a Docker image containing Jupyter Lab preconfigured with .NET 8 support, allowing me to easily create and run .NET code blocks in a notebook.

Thanks to this Docker image, I can now quickly spin up a Jupyter Lab environment with full .NET support, without having to create a C# project in Visual Studio or VS Code.

# The Dockerfile

Here is the content I defined for the Dockerfile.

It is of course available [in this dedicated GitHub repository](https://github.com/agailloty/jupyter-dotnet).

```Dockerfile
# Use a base image with Jupyter Lab and .NET 8 support
FROM jupyter/base-notebook:latest

# Switch to the root user to install packages without sudo
USER root

# Install the .NET 8.0 SDK
RUN wget https://packages.microsoft.com/config/ubuntu/20.04/packages-microsoft-prod.deb -O packages-microsoft-prod.deb && \
    dpkg -i packages-microsoft-prod.deb && \
    apt-get update && \
    apt-get install -y apt-transport-https && \
    apt-get update && \
    apt-get install -y dotnet-sdk-8.0

# Install the .NET interactive tools
RUN dotnet tool install --global Microsoft.dotnet-interactive

# Add the .NET interactive tools to the PATH
ENV PATH="${PATH}:~/.dotnet/tools"

# Install Jupyter notebooks for .NET
RUN dotnet interactive jupyter install

# Expose the Jupyter Lab port
EXPOSE 8888

# Start Jupyter Lab
CMD ["jupyter", "lab", "--ip=0.0.0.0", "--allow-root"]
```

Explanation of the different parts:

- **FROM jupyter/base-notebook:latest**: Base Jupyter Lab image with .NET 8 support.
- **USER root**: Switch to the root user for administration commands.
- **RUN ...**: Install the .NET 8.0 SDK and the .NET interactive tools.
- **ENV PATH**: Add the .NET interactive tools to the PATH.
- **RUN dotnet interactive jupyter install**: Install Jupyter notebooks for .NET.
- **EXPOSE 8888**: Expose port 8888 for Jupyter Lab.
- **CMD**: Start Jupyter Lab.

If you are curious to try this Docker image yourself, you can find it on [my dedicated GitHub repository](https://github.com/agailloty/jupyter-dotnet). Feel free to test it and share your comments or suggestions for improvement.

# Conclusion

In a world where speed and efficiency are essential, having tools that let you iterate quickly and test new ideas is a valuable asset for any developer. With this Docker image, I found a simple and effective way to do exactly that, and I hope it will also be useful to other programming enthusiasts in their learning and exploration journey.
