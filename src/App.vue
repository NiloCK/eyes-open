<template>
    <div class="max-w-7xl mx-auto p-6">
        <!-- hidden canvas for conversion / rendering -->
        <canvas ref="canvas" style="display: none"></canvas>

        <h1 class="text-2xl font-bold mb-6">Claude Eyes-Open SVG Drawing</h1>

        <form @submit.prevent="handleSubmit" class="space-y-4">
            <div>
                <label class="block mb-2">API Key:</label>
                <input
                    type="password"
                    v-model="apiKey"
                    class="w-full p-2 border rounded"
                    required
                />
            </div>

            <div>
                <label class="block mb-2">Scene Description:</label>
                <textarea
                    v-model="message"
                    class="w-full p-2 border rounded h-32"
                    placeholder="Describe the scene you want Claude to draw..."
                    required
                />
            </div>

            <button
                type="submit"
                :disabled="loading"
                class="bg-blue-500 text-white px-4 py-2 rounded disabled:bg-blue-300"
            >
                {{ loading ? "Sending..." : "Send to Claude" }}
            </button>
        </form>

        <div v-if="error" class="mt-4 p-4 bg-red-100 text-red-700 rounded">
            {{ error }}
        </div>

        <div v-if="response" class="mt-4">
            <h2 class="font-bold mb-2">Claude's Response:</h2>
            <div class="p-4 bg-gray-100 rounded whitespace-pre-wrap">
                {{ response }}
            </div>
        </div>
        <div v-if="extractedSvg" class="mt-4">
            <h2 class="font-bold mb-2">Generated SVG:</h2>
            <div
                class="p-4 bg-white border rounded shadow-sm"
                v-html="extractedSvg"
            ></div>
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
                                    <span class="mr-2 text-sm text-gray-600">
                                        {{
                                            item.showRaw
                                                ? "Raw SVG"
                                                : "Rendered"
                                        }}
                                    </span>
                                    <button
                                        @click="item.showRaw = !item.showRaw"
                                        class="px-3 py-1 text-sm bg-gray-200 hover:bg-gray-300 rounded-full transition-colors"
                                    >
                                        Toggle View
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
                                class="p-4 bg-gray-50 border rounded h-full flex items-center justify-center text-gray-500"
                            >
                                No image in this response
                            </div>
                        </div>
                    </div>
                </div>
            </div>
        </div>

        <div v-if="extractedSvg" class="mt-4">
            <button
                @click="continueWithRenderedImage"
                :disabled="loading"
                class="bg-green-500 text-white px-4 py-2 rounded disabled:bg-green-300"
            >
                {{ loading ? "Processing..." : "Continue with this result" }}
            </button>
        </div>
    </div>
</template>

<script setup>
import { ref, watch } from "vue";
import Anthropic from "@anthropic-ai/sdk";

const apiKey = ref("");
const message = ref("");
const currentImage = ref(null);
const response = ref("");
const loading = ref(false);
const error = ref("");
const extractedSvg = ref("");
const canvas = ref(null);
const conversationHistory = ref([]);
const responseHistory = ref([]);

const svgToImage = () => {
    return new Promise((resolve, reject) => {
        const svg = document.querySelector("svg");
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
        message.value = "";

        await handleSubmit();
    } catch (err) {
        error.value = "Error converting SVG to image: " + err.message;
    } finally {
        loading.value = false;
    }
};

const handleImageChange = (e) => {
    const file = e.target.files[0];
    if (file) {
        const reader = new FileReader();
        reader.onloadend = () => {
            currentImage.value = reader.result;
        };
        reader.readAsDataURL(file);
    }
};

watch(response, (newResponse) => {
    if (newResponse) {
        const svg = extractSvgFromResponse(newResponse);
        extractedSvg.value = svg;

        const textMinusSvg = newResponse.replace(svg, "");

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
                        text: basePrompt + message.value,
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
