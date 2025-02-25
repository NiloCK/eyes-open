# Claude Eyes-Open SVG Drawing Tool

A Vue app that enables Claude to create SVG drawings with visual feedback, allowing it to see and improve its artwork iteratively.

Deployed at https://nilock.github.com/eyes-open

## Features

- Use your Anthropic API key with various Claude models
- Customize system prompts
- View both text reasoning and SVG output
- Toggle between rendered and raw SVG code
- Export sessions as HTML or Markdown

## Getting Started

1. Clone repository and install dependencies
2. Run `npm run dev` or `yarn dev`
3. Enter your Anthropic API key to begin

## How It Works

You describe a scene, and Claude draws it using SVG. After each step, Claude sees its rendered output and makes improvements based on visual feedback until satisfied.

## Requirements

- Anthropic API key with access to vision-capable Claude models
- Modern web browser
- Node.js and npm/yarn
