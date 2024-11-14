<template>
    <div class="max-w-7xl mx-auto p-6">
        <!-- hidden canvas for conversion / rendering -->
        <canvas ref="canvas" style="display: none"></canvas>

        <h1 class="text-2xl font-bold mb-6">Claude Eyes-Open SVG Drawing</h1>

        <form @submit.prevent="handleSubmit" class="space-y-4">
            <div>
                <label class="block mb-2">Anthropic API Key:</label>
                <input
                    type="password"
                    v-model="apiKey"
                    class="w-full p-2 border rounded"
                    placeholder="..."
                    required
                    :disabled="started"
                />
            </div>

            <div>
                <label class="block mb-2">System Prompt:</label>
                <textarea
                    v-model="system"
                    class="w-full p-2 border rounded h-32"
                    placeholder=""
                    required
                    :disabled="started"
                />
            </div>

            <div>
                <label class="block mb-2">Scene Description:</label>
                <textarea
                    v-model="sceneInput"
                    class="w-full p-2 border rounded h-32"
                    placeholder="Describe the scene you want Claude to draw..."
                    required
                    :disabled="started"
                />
            </div>

            <button
                type="submit"
                :disabled="loading || started"
                class="bg-blue-500 text-white px-4 py-2 rounded disabled:bg-blue-300"
            >
                {{ loading ? "Sending..." : "Start Drawing" }}
            </button>
        </form>

        <div v-if="error" class="mt-4 p-4 bg-red-100 text-red-700 rounded">
            {{ error }}
        </div>

        <div class="mt-8">
            <h2 class="text-xl font-bold mb-4">Drawing Progress:</h2>
            <div class="space-y-6">
                <div
                    v-for="(item, index) in responseHistory"
                    :key="index"
                    class="border rounded p-4"
                >
                    <div class="font-bold mb-2">Step {{ index + 1 }}</div>
                    <div class="flex gap-4">
                        <!-- Text response -->
                        <div
                            class="w-1/3 p-4 bg-gray-100 rounded whitespace-pre-wrap overflow-auto max-h-[500px]"
                        >
                            {{ item.text }}
                        </div>
                        <!-- SVG display -->
                        <div class="w-2/3">
                            <div v-if="item.svg" class="space-y-2">
                                <!-- Toggle switch -->
                                <div class="flex items-center justify-end mb-2">
                                    <button
                                        @click="item.showRaw = !item.showRaw"
                                        class="px-3 py-1 text-sm bg-gray-200 hover:bg-gray-300 rounded-full transition-colors"
                                    >
                                        {{
                                            item.showRaw
                                                ? "Show 🖼️"
                                                : "Show </>"
                                        }}
                                    </button>
                                </div>

                                <!-- Raw SVG code -->
                                <pre
                                    v-if="item.showRaw"
                                    class="p-4 bg-gray-100 rounded overflow-auto text-sm font-mono text-left"
                                    style="text-align: left"
                                    >{{ item.svg }}</pre
                                >

                                <!-- Rendered SVG -->
                                <div
                                    v-else
                                    class="p-4 bg-white border rounded svg-container"
                                    v-html="item.svg"
                                ></div>
                            </div>
                            <div
                                v-else
                                class="p-4 bg-gray-50 border rounded h-full flex items-center justify-center text-gray-500 gap-4"
                            >
                                <div>No image in this response - all done.</div>

                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div v-if="extractedSvg" class="mt-4 space-x-3">
            <button
                @click="continueWithRenderedImage"
                :disabled="loading"
                class="bg-green-500 text-white px-4 py-2 rounded disabled:bg-green-300 hover:bg-green-600 transition-colors"
            >
                {{ loading ? "Processing..." : "Iterate on this result" }}
            </button>
            <button
                @click="exportSessionHTML"
                class="bg-purple-500 text-white px-4 py-2 rounded hover:bg-purple-600 transition-colors"
            >
                Export as HTML 📁
            </button>
            <button
                @click="exportSessionMD"
                class="bg-purple-500 text-white px-4 py-2 rounded hover:bg-purple-600 transition-colors"
            >
                Copy MD 📋
            </button>
        </div>
    </div>
</template>

<script setup>
import { ref, watch } from "vue";
import Anthropic from "@anthropic-ai/sdk";

const apiKey = ref("");
const sceneInput = ref("");
const currentImage = ref(null);
const response = ref("");
const loading = ref(false);
const started = ref(false);
const error = ref("");
const extractedSvg = ref("");
const canvas = ref(null);
const conversationHistory = ref([]);
const responseHistory = ref([]);
const system = ref(
  `You are a talented and pragmatic artist, capable of creating images via SVG.

You will receive a single user prompt for a scene, after which point you will iteratively create the scene.

Each of your responses will include a single <svg> delineated image.

Each of your SVGs will be rendered and returned to you as an image by your patron - the user.

Each of your responses should include a description of the intended effect of the SVG, as well as an evaluation of the prior render against its own described goals.

You are free to compartmentalize your drawing efforts - maybe it is useful to draw individual elements separately, and later combine. Maybe it is useful to draw a background first, and then overlay other elements.

Again: you are pragmatic. You aim for incremental progress instead of perfection.

When a returned render satisfies your expectations, please respond with a message that does not include an SVG, to indicate that you are finished.`);


const escapeHTML = (str) => {
  return str.replace(/[&<>'"]/g, (tag) => ({
    "&": "&amp;",
    "<": "&lt;",
    ">": "&gt;",
    "'": "&#39;",
    '"': "&quot;",
  }[tag] || tag));
}

const exportSessionMD = () => {
  const mdContent = `### ${sceneInput.value}
Prompt:

${system.value.split('\n').map(l =>
  ">  " +escapeHTML(l))
.join('\n')}

Scene:

>  ${sceneInput.value}

| Commentary | Output |
| --- | --- |
${responseHistory.value.map((item) =>
    `| ${item.text.replace(/\n/g, ' ')} | ${item.svg ? item.svg.replace(/\n/g, ' ') : 'No image'} |`
  ).join('\n')}`;

  // copy mdContent to clipboard
  navigator.clipboard.writeText(mdContent);

  alert("Markdown content copied to clipboard!");
}

const exportSessionHTML = () => {



    // Create HTML content
    const htmlContent = `
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Claude Eyes-Open SVG Drawing Session Export</title>
    <style>
        body {
            max-width: 1200px;
            margin: 0 auto;
            padding: 20px;
            font-family: system-ui, -apple-system, sans-serif;
        }
        .response {
            border: 1px solid #ccc;
            margin: 20px 0;
            padding: 20px;
            border-radius: 8px;
        }
        .text-content {
            background: #f3f4f6;
            padding: 15px;
            border-radius: 8px;
            margin-bottom: 15px;
            white-space: pre-wrap;
        }
        .svg-content {
            background: white;
            padding: 15px;
            border: 1px solid #eee;
            border-radius: 8px;
        }
        pre {
            white-space: pre-wrap;
            background: #f8f8f8;
            padding: 10px;
            border-radius: 4px;
        }
    </style>
</head>
<body>
    <h1>Claude Eyes-Open SVG Drawing Session</h1>

    <h2>System Prompt:</h2>
    <pre>${escapeHTML(system.value)}</pre>

    <h2>Scene Description:</h2>
    <pre>${sceneInput.value}</pre>

    <h2>Drawing Progress:</h2>
    ${responseHistory.value
        .map(
            (item, index) => `
    <div class="response">
        <h3>Step ${index + 1}</h3>
        <div class="text-content">${item.text}</div>
        ${
            item.svg
                ? `<div class="svg-content">${item.svg}</div>`
                : '<div>No image in this response</div>'
        }
    </div>
    `
        )
        .join('')}
</body>
</html>
    `;

    // Create blob and download
    const blob = new Blob([htmlContent], { type: 'text/html' });
    const url = window.URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `claude-draw-${sceneInput.value.replaceAll(" ", "_")}.html`;
    document.body.appendChild(a);
    a.click();
    window.URL.revokeObjectURL(url);
    document.body.removeChild(a);
};

const svgToImage = () => {
    return new Promise((resolve, reject) => {
        const svgs = document.querySelectorAll("svg");
        if (svgs.length === 0) {
            reject("No SVG found in the response");
        }
        const svg = svgs[svgs.length - 1];

        const serializer = new XMLSerializer();
        const svgStr = serializer.serializeToString(svg);

        const svgBlob = new Blob([svgStr], {
            type: "image/svg+xml;charset=utf-8",
        });
        const URL = window.URL || window.webkitURL || window;
        const blobURL = URL.createObjectURL(svgBlob);

        const img = new Image();
        img.onload = () => {
            const ctx = canvas.value.getContext("2d");

            // Set canvas size to match SVG
            canvas.value.width = img.width;
            canvas.value.height = img.height;

            // Draw white background
            ctx.fillStyle = "white";
            ctx.fillRect(0, 0, canvas.value.width, canvas.value.height);

            // Draw the image
            ctx.drawImage(img, 0, 0);

            // Get base64 without the prefix
            const dataUrl = canvas.value.toDataURL("image/jpeg", 1.0);
            const base64Data = dataUrl.split(",")[1];
            URL.revokeObjectURL(blobURL);
            resolve(base64Data); // Return only the base64 data without the prefix
        };

        img.onerror = () => {
            URL.revokeObjectURL(blobURL);
            reject(new Error("Error loading SVG"));
        };

        img.src = blobURL;
    });
};

const continueWithRenderedImage = async () => {
    try {
        loading.value = true;
        const base64Data = await svgToImage();

        // Construct the full data URL for preview
        currentImage.value = `data:image/jpeg;base64,${base64Data}`;

        await handleSubmit();
    } catch (err) {
        error.value = "Error converting SVG to image: " + err.message;
    } finally {
        loading.value = false;
    }
};

watch(response, (newResponse) => {
    if (newResponse) {
        const svg = extractSvgFromResponse(newResponse);
        extractedSvg.value = svg;

        const textMinusSvg = newResponse.replace(svg, "[clipped SVG]");

        if (svg) {
            // Add new response to history with showRaw property
            responseHistory.value.push({
                response: newResponse,
                svg: svg,
                text: textMinusSvg,
                showRaw: false, // Add this line
            });
        } else {
            responseHistory.value.push({
                response: newResponse,
                svg: null,
                text: newResponse,
                showRaw: false, // Add this line
            });
            console.log("No SVG found in response");
        }
    }
});

const extractSvgFromResponse = (responseText) => {
    const svgMatch = responseText.match(/<svg[\s\S]*?<\/svg>/);
    return svgMatch ? svgMatch[0] : null;
};

const basePrompt = `You are capable of creating images via SVG, but blind-drawing has limitations.

Let's see how visual feedback can help your drawing skills.

At the end of this prompt, I will ask for a specific scene.

After this prompt, I will respond *only* with the rendered images of your SVGs, so that you can view your work. This will continue until you return a response with no SVGs. Please do so to indicate that you are finished and satisfied with the result.

Please return a single SVG in each response, wrapped in an <svg> tag. Outside the svg tag, you can include descriptions or running commentary on your process.

I'll encourage you to compartmentalize your drawing efforts - maybe it is useful to draw individual elements separately, and later combine. Maybe it is useful to draw a background first, and then overlay other elements.

The scene:

`;

const handleSubmit = async () => {
    loading.value = true;
    started.value = true;
    error.value = "";
    response.value = "";

    try {
        const anthropic = new Anthropic({
            apiKey: apiKey.value,
            dangerouslyAllowBrowser: true, // Enable browser usage,
        });

        // Initialize conversation if this is the first message
        if (conversationHistory.value.length === 0) {
            conversationHistory.value.push({
                role: "user",
                content: [
                    {
                        type: "text",
                        text: sceneInput.value,
                    },
                ],
            });
        }

        // If there's a current image, add it to the latest message
        const newMessage = {
            role: "user",
            content: [],
        };

        if (currentImage.value) {
            newMessage.content.push({
                type: "image",
                source: {
                    type: "base64",
                    media_type: "image/jpeg",
                    data: currentImage.value.split(",")[1],
                },
            });
        }

        // Only add the current message if it has content
        if (newMessage.content.length > 0) {
            conversationHistory.value.push(newMessage);
        }

        const completion = await anthropic.messages.create({
            model: "claude-3-5-sonnet-latest",
            max_tokens: 8192,
            messages: conversationHistory.value,
            system: system.value,
        });

        response.value = completion.content[0].text;

        // Add Claude's response to the conversation history
        conversationHistory.value.push({
            role: "assistant",
            content: [
                {
                    type: "text",
                    text: response.value,
                },
            ],
        });
    } catch (err) {
        error.value =
            err.message || "An error occurred while communicating with Claude";
    } finally {
        loading.value = false;
    }
};
</script>

<style scoped>
.svg-container {
    display: flex;
    justify-content: center;
    align-items: center;
}

:deep(svg) {
    width: 100%;
    height: auto;
    display: block;
}

pre {
    max-height: 500px;
    white-space: pre-wrap;
}

:deep(svg) {
    max-height: 500px;
}

/* Ensure the text container doesn't get too wide */
.whitespace-pre-wrap {
    word-break: break-word;
}
</style>
