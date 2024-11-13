<template>
    <div class="max-w-2xl mx-auto p-6">
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

            <!-- <div>
                <label class="block mb-2">Image (optional):</label>
                <input
                    type="file"
                    @change="handleImageChange"
                    accept="image/*"
                    class="w-full"
                    ref="fileInput"
                />
                <img
                    v-if="image"
                    :src="image"
                    alt="Selected"
                    class="mt-2 max-h-48 object-contain"
                />
            </div> -->

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
    </div>
</template>

<script setup>
import { ref, watch } from "vue";
import Anthropic from "@anthropic-ai/sdk";

const apiKey = ref("");
const message = ref("");
const image = ref(null);
const response = ref("");
const loading = ref(false);
const error = ref("");
const fileInput = ref(null);
const extractedSvg = ref("");

const handleImageChange = (e) => {
    const file = e.target.files[0];
    if (file) {
        const reader = new FileReader();
        reader.onloadend = () => {
            image.value = reader.result;
        };
        reader.readAsDataURL(file);
    }
};

watch(response, (newResponse) => {
    if (newResponse) {
        const svg = extractSvgFromResponse(newResponse);
        extractedSvg.value = svg;

        if (!svg) {
            console.log("No SVG found in response");
        }
    } else {
        extractedSvg.value = "";
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
            dangerouslyAllowBrowser: true, // Enable browser usage
        });

        const combinedMessage = basePrompt + message.value;

        const messages = [
            {
                role: "user",
                content: [{ type: "text", text: combinedMessage }],
            },
        ];

        // // Add image to messages if one is selected
        // if (image.value) {
        //     messages[0].content.unshift({
        //         type: "image",
        //         source: {
        //             type: "base64",
        //             media_type: "image/jpeg",
        //             data: image.value.split(",")[1], // Remove the data:image/jpeg;base64, prefix
        //         },
        //     });
        // }

        const completion = await anthropic.messages.create({
            model: "claude-3-5-sonnet-latest",
            max_tokens: 8192,
            messages: messages,
        });

        response.value = completion.content[0].text;
    } catch (err) {
        error.value =
            err.message || "An error occurred while communicating with Claude";
    } finally {
        loading.value = false;
    }
};
</script>
