---
pcx_content_type: reference
title: Deploy a Constellation Worker
meta:
    title: Deploy a Constellation Worker
weight: 1
---

# Make your first Constellation Worker

In this guide, you will build an [image classification](https://developers.google.com/machine-learning/practica/image-classification#how_image_classification_works) application powered by a Constellation inference engine and the [SqueezeNet 1.1](https://github.com/onnx/models/blob/main/vision/classification/squeezenet/README.md) ONNX model. SqueezeNet is a small Convolutional Neural Network (CNN) which achieves AlexNet-level accuracy on ImageNet with 50 times fewer parameters.

## Configure your project

Generate your new project by running [`create`](/constellation/platform/wrangler/#manage-projects). Then run `list` to review th details of your newly created `image-classifier` project:

```bash
$ npx wrangler constellation project create "image-classifier" ONNX
$ npx wrangler constellation project list

┌──────────────────────────────────────┬──────────────────────┬─────────┐
│ id                                   │ name                 │ runtime │
├──────────────────────────────────────┼──────────────────────┼─────────┤
│ 2193053a-af0a-40a6-b757-00fa73908ef6 │ image-classifier     │ ONNX    │
└──────────────────────────────────────┴──────────────────────┴─────────┘
```

Next, create a new Worker project. We are going to need [Wrangler](/constellation/platform/wrangler/).

```bash
$ mkdir image-classifier-worker
$ cd image-classifier-worker
$ npm init -f
$ npm install wrangler@beta --save-dev
$ npx wrangler init
```

Follow the prompts. We are going to skip tests for now.

```bash
Would you like to use git to manage this Worker?: N
Would you like to install the type definitions for Workers into your package.json?: Y
Would you like to create a Worker at src/index.ts?: Fetch handler
Would you like us to write your first test with Vitest?: N
```

You can learn more on how to start new projects with Wrangler [here](/workers/get-started/guide/).

In your folder, you should now find a [wrangler.toml](/workers/wrangler/configuration/) file.

Now add the Constellation configuration to the wrangler.toml configuration file with the project [binding](/constellation/platform/wrangler/#bindings):

```toml
---
filename: wrangler.toml
---
# Top-level configuration
name = "image-classifier"
main = "src/index.ts"
node_compat = true
workers_dev = true
compatibility_date = "2023-05-14"

constellation = [
    {binding = 'CLASSIFIER', project_id = '2193053a-af0a-40a6-b757-00fa73908ef6'},
]
```

Make sure to substitute the `project_id` with the one enumerated when you ran `npx wrangler constellation project list`.

In your `image-classifier` directory, install the client API library:

```bash
$ npm install @cloudflare/constellation --save-dev
```

## Upload model

Upload the pre-trained [SqueezeNet 1.1](https://github.com/onnx/models/blob/main/vision/classification/squeezenet/README.md) ONNX model to your project:

```bash
$ wget https://github.com/microsoft/onnxjs-demo/raw/master/docs/squeezenet1_1.onnx
$ npx wrangler constellation model upload "image-classifier" "squeezenet11" squeezenet1_1.onnx
$ npx wrangler constellation model list "image-classifier"

┌──────────────────────────────────────┬──────────────────────────────────────┬──────────────┐
│ id                                   │ project_id                           │ name         │
├──────────────────────────────────────┼──────────────────────────────────────┼──────────────┤
│ 297f3cda-5e55-33c0-8ffe-224876a76a39 │ 2193053a-af0a-40a6-b757-00fa73908ef6 │ squeezenet11 │
└──────────────────────────────────────┴──────────────────────────────────────┴──────────────┘
```

Take note of the id field as this will be the model id.

## Download Imagenet classes

The SqueezeNet model was trained on top of the [Imagenet](https://www.image-net.org/) dataset. Make a new `src` folder in your `image-classifier` directory. Then download the the list of 1,000 image classes that SqueezeNet was trained for:

```bash
$ mkdir src
$ wget -O src/imagenet.ts \
  https://raw.githubusercontent.com/microsoft/onnxjs-demo/master/src/data/imagenet.ts
```

## Install modules

Install [pngjs](https://github.com/pngjs/pngjs), a PNG decoder, and [string-to-stream](https://github.com/feross/string-to-stream) before you begin coding your project:

```bash
$ npm install string-to-stream --save-dev
$ npm install pngjs --save-dev
```

## Code

With your project configured, begin coding. The following script gets a PNG file upload from the request, decodes the image to RGB raw bitmaps, constructs a 3D tensor with the input data, runs the SqueezeNet model, maps the top predictions to the ImagetNet human-readable classes and returns the strongest one in a JSON object.

Replace <code>297f3cda-5e55-33c0-8ffe-224876a76a39</code> with your actual model ID.

```javascript
---
filename: src/index.ts
---
import str from "string-to-stream";
import { PNG } from "pngjs/browser";

import { imagenetClasses } from "./imagenet";
import { Tensor, run } from "@cloudflare/constellation";

export default {
    async fetch(request: Request, env: Env): Promise<Response> {
        const formData = await request.formData();
        const file = formData.get("file") as unknown as File;
        if (file) {
            const data = await file.arrayBuffer();
            const result = await processImage(env, data);
            return new Response(JSON.stringify(result));
        } else {
            return new Response("nothing to see here");
        }
    },
};

async function processImage(env: Env, data: ArrayBuffer) {
    let result;

    const input = await decodeImage(data).catch((err) => {
        result = err;
    });

    if (input) {
        const tensorInput = new Tensor("float32", [1, 3, 224, 224], input);

        const output = await run(
            env.CLASSIFIER,
            "297f3cda-5e55-33c0-8ffe-224876a76a39",
            tensorInput
        );

        const predictions = output.squeezenet0_flatten0_reshape0.value;
        const softmaxResult = softmax(predictions);
        const results = topClasses(softmaxResult, 5);

        result = results[0];
    }

    return result;
}

/* The model expects input images normalized in the same way, i.e. mini-batches of 3-channel RGB images
   of shape (N x 3 x H x W), where N is the batch size, and H and W are expected to be 224. */

async function decodeImage(
    buffer: ArrayBuffer,
    width: number = 224,
    height: number = 224
): Promise<any> {
    return new Promise(async (ok, err) => {
        // convert string to stream
        const stream: any = str(buffer as unknown as string);

        stream
            .pipe(
                new PNG({
                    filterType: 4,
                })
            )
            .on("parsed", function (this: any) {
                if (this.width != width || this.height != height) {
                    err({
                        err: `expected width to be ${width}x${height}, given ${this.width}x${this.height}`,
                    });
                } else {
                    const [redArray, greenArray, blueArray] = new Array(
                        new Array<number>(),
                        new Array<number>(),
                        new Array<number>()
                    );

                    for (let i = 0; i < this.data.length; i += 4) {
                        redArray.push(this.data[i] / 255.0);
                        greenArray.push(this.data[i + 1] / 255.0);
                        blueArray.push(this.data[i + 2] / 255.0);
                        // skip data[i + 3] to filter out the alpha channel
                    }

                    const transposedData = redArray
                        .concat(greenArray)
                        .concat(blueArray);
                    ok(transposedData);
                }
            })
            .on("error", function (error: any) {
                err({ err: error.toString() });
            });
    });
}

// See https://en.wikipedia.org/wiki/Softmax_function
// Transforms values to between 0 and 1
// The sum of all outputs generated by softmax is 1.

function softmax(resultArray: number[]): any {
    const largestNumber = Math.max(...resultArray);
    const sumOfExp = resultArray
        .map((resultItem) => Math.exp(resultItem - largestNumber))
        .reduce((prevNumber, currentNumber) => prevNumber + currentNumber);
    return resultArray.map((resultValue) => {
        return Math.exp(resultValue - largestNumber) / sumOfExp;
    });
}

/* Get the top n classes from ImagetNet */

export function topClasses(classProbabilities: any, n = 5) {
    const probabilities = ArrayBuffer.isView(classProbabilities)
        ? Array.prototype.slice.call(classProbabilities)
        : classProbabilities;

    const sorted = probabilities
        .map((prob: any, index: number) => [prob, index])
        .sort((a: Array<number>, b: Array<number>) => {
            return a[0] == b[0] ? 0 : a[0] > b[0] ? -1 : 1;
        });

    const top = sorted.slice(0, n).map((probIndex: Array<number>) => {
        const iClass = imagenetClasses[probIndex[1]];
        return {
            id: iClass[0],
            index: parseInt(probIndex[1].toString(), 10),
            name: iClass[1].replace(/_/g, " "),
            probability: probIndex[0],
        };
    });

    return top;
}

export interface Env {
    CLASSIFIER: any;
}

```

## Test your project

### Download test images

Below are some test 224x244 PNG images you can use for tests:

```bash
$ wget https://imagedelivery.net/WPOeHKUnTTahhk4F5twuvg/8b78a6fb-44ac-4a97-121b-fb8f47f1e000/public -O cat.png
$ wget https://imagedelivery.net/WPOeHKUnTTahhk4F5twuvg/05c265ae-d3c0-4114-208b-a2d7709cc100/public -O house.png
$ wget https://imagedelivery.net/WPOeHKUnTTahhk4F5twuvg/4152ee23-f9af-4b21-a636-600e33883400/public -O mountain.png
```

### Run `wrangler dev`

Start a local server to test your `image-classifier` Worker by running [`wrangler dev`](/workers/wrangler/commands/#dev):

```bash
$ npx wrangler dev
⬣ Listening at http://0.0.0.0:8787

```bash
$ curl http://0.0.0.0:8787 -F file=@cat.png
{"id":"n02124075","index":285,"name":"Egyptian cat","probability":0.5356272459030151}

$ curl http://0.0.0.0:8787 -F file=@house.png
{"id":"n03028079","index":497,"name":"church","probability":0.5730999112129211}

$ curl http://0.0.0.0:8787 -F file=@mountain.png
{"id":"n09246464","index":972,"name":"cliff","probability":0.37886714935302734}
```

This is it, your image classifier is working. Run it through other 224x244 PNG images of your own and check the results.

## Publish your projct

When you are ready, deploy your Worker:

```bash
$ npx wrangler publish