# Epic Next JS 14 TutorialPart 6: Create Video Summary with Next.js, LangChain, and Open AI.

In the previous tutorial, we completed our **Dashboard** and **Account** pages. In this section, we will work on generating our video summary using Open AI and LangChain.

- Getting Started with the Project
- Building Out The Hero Section
- Building Out The Features Section, TopNavigation and Footer
- Building Out the Register and Sign In Page
- Building Out the Dashboard page
- **Get Video Transcript with OpenAI Function**
- Strapi CRUD permissions
- Search & pagination
- Backend deployment to Strapi Cloud and frontend deployment to Vercel

![001-summary-form](./images/001-summary-form.png)

We will kick off the tutorial by working on our SummaryForm component. This time around, instead of using a `server action,` we will explore how to create an api route in Next.js.

You can learn more about Next.js route handlers [here](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)

But first, let's create our Summary Form, which we can use to submit the request.

Navigate to `src/components/forms` and create a new file called `SummaryForm.tsx` and paste in the following code as the starting point.

```tsx
"use client";

import { useState } from "react";
import { toast } from "sonner";
import { cn } from "@/lib/utils";

import { Input } from "@/components/ui/input";
import { SubmitButton } from "@/components/custom/SubmitButton";

interface StrapiErrorsProps {
  message: string | null;
  name: string;
}

const INITIAL_STATE = {
  message: null,
  name: "",
};

export function SummaryForm() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<StrapiErrorsProps>(INITIAL_STATE);
  const [value, setValue] = useState<string>("");

  async function handleFormSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    setLoading(true);
    toast.success("Summary Created");
    setLoading(false);
  }

  function clearError() {
    setError(INITIAL_STATE);
    if (error.message) setValue("");
  }

  const errorStyles = error.message
    ? "outline-1 outline outline-red-500 placeholder:text-red-700"
    : "";

  return (
    <div className="w-full max-w-[960px]">
      <form
        onSubmit={handleFormSubmit}
        className="flex gap-2 items-center justify-center"
      >
        <Input
          name="videoId"
          placeholder={
            error.message ? error.message : "Youtube Video ID or URL"
          }
          value={value}
          onChange={(e) => setValue(e.target.value)}
          onMouseDown={clearError}
          className={cn(
            "w-full focus:text-black focus-visible:ring-pink-500",
            errorStyles
          )}
          required
        />

        <SubmitButton
          text="Create Summary"
          loadingText="Creating Summary"
          loading={loading}
        />
      </form>
    </div>
  );
}
```

The above code contains a basic form UI and a `handleFormSubmit` function, which does not include any of our logic to get the summary yet.

We also use **Sonner**, one of my favorite toast libraries. You can learn more about it [here](https://sonner.emilkowal.ski/).

But we are not using it directly; instead, we are using the **Chadcn UI** component, which you can find [here](https://ui.shadcn.com/docs/components/sonner).

```bash
npx shadcn-ui@latest add sonner
```

Once **Sonner** is installed, implement it in our main `layout.tsx` file by adding the following import.

```tsx
import { Toaster } from "@/components/ui/sonner";
```

And adding the code below above our `TopNav`.

```tsx
<body className={inter.className}>
  <Toaster position="bottom-center" />
  <Header data={globalData.header} />
  <div>{children}</div>
  <Footer data={globalData.footer} />
</body>
```

Let's add this form to our top navigation by navigating to the `src/components/custom/Header.tsx` file and making the following changes.

```tsx
// import the form
import { SummaryForm } from "@/components/forms/SummaryForm";

// rest of the code

export async function Header({ data }: Readonly<HeaderProps>) {
  const user = await getUserMeLoader();
  const { logoText, ctaButton } = data;
  return (
    <div className="flex items-center justify-between px-4 py-3 bg-white shadow-md dark:bg-gray-800">
      <Logo text={logoText.text} />
      {user.ok && <SummaryForm />} {/* --> add this */}
      <div className="flex items-center gap-4">
        {user.ok ? (
          <LoggedInUser userData={user.data} />
        ) : (
          <Link href={ctaButton.url}>
            <Button>{ctaButton.text}</Button>
          </Link>
        )}
      </div>
    </div>
  );
}
```

Let's restart our frontend project and see if it shows up.

![002-toast-and-form](./images/002-toast-and-form.gif)

Now that our basic form is working let's examine how to set up our first API Handler Route in Next.js.

## How To Create A Route Handler in Next.js

We will have the Next.js [docs](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) open as a reference.

Let's start by creating a new folder inside our `app` directory called `api`, a folder called `summarize`, and a file called `route.ts`. Then, paste in the following code.

```ts
import { NextRequest } from "next/server";

export async function POST(req: NextRequest) {
  console.log("FROM OUR ROUTE HANDLER:", req.body);
  try {
    return new Response(
      JSON.stringify({ data: "return from our handler", error: null }),
      {
        status: 200,
      }
    );
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }
}
```

Next, let's create a service to call our new route handler. Navigate to `src/data/services` and create a new file called `summary-service.ts`.

Create a new service called `generateSummaryService.ts` and add the following code.

```ts
export async function generateSummaryService(videoId: string) {
  const url = "/api/summarize";
  try {
    const response = await fetch(url, {
      method: "POST",
      body: JSON.stringify({ videoId: videoId }),
    });
    return await response.json();
  } catch (error) {
    console.error("Failed to generate summary:", error);
    if (error instanceof Error) return { error: { message: error.message } };
    return { data: null, error: { message: "Unknown error" } };
  }
}
```

The following service allows us to call our newly created route handler located at `api/summarize` endpoint. It expects us to pass a `videoId` for the video we want to summarize.

Now that we have our basic route handler let's return to our `SummaryForm.tsx` file and see if we can request this endpoint.

Let's modify our `handleFormSubmit` with the following code to use our newly created service. Don't forget to import the `generateSummaryService` service.

```tsx
import { generateSummaryService } from "@/data/services/summary-service";
```

```tsx
async function handleFormSubmit(event: React.FormEvent<HTMLFormElement>) {
  event.preventDefault();
  setLoading(true);

  const formData = new FormData(event.currentTarget);
  const videoId = formData.get("videoId") as string;

  console.log(videoId);

  const summaryResponseData = await generateSummaryService(videoId);
  console.log(summaryResponseData, "Response from route handler");

  toast.success("Testing Toast");
  setLoading(false);
}
```

When you submit your form, you should see the message returned from our route handler in our console log.

![003-route-response](./images/003-route-response.gif)

Now that we know that our **Summary Form** and **Route Handler** are connected, we can work on the logic responsible for summarizing our video.

## Using Next.js Route Handler LangChain and Open AI To Create A Summary.

This section will examine how to create a video summary based on the video transcript.

We will be using a couple of services to help us accomplish this.

Prerequisite: You must have an account with Open AI. If you don't have one, go [here](https://platform.openai.com/docs/introduction) and create one.

### Getting Transcript From YouTube

In the past, I have used the ['youtube-transcript'](https://www.npmjs.com/package/youtube-transcript) library; interestingly, it broke just recently.

You can read through the issues [here](https://github.com/Kakulukian/youtube-transcript/issues/19#issuecomment-2041204365).

Luckily, the fantastic community created a workaround, which we will use in our code.

The fix might have been merged for all who read this in the future, so you could update your code and use the `youtube-transcript` package directly.

A quick note: Using other libraries, especially ones not officially supported, can lead to breaking changes, so keep this in mind.

So, if you need a service to be guaranteed, it may be valuable to create your own implementation.

I started building a plugin in Strapi to create transcripts using the OpenAi whisper service. Once it is done, I will make a blog post around it.

But for this video, we will follow the following steps: navigate to `src/lib`, create a file named `youtube-transcript.ts`, and paste it into the following code.

```ts
import { parse } from "node-html-parser";

const USER_AGENT =
  "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_4) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/85.0.4183.83 Safari/537.36,gzip(gfe)";

class YoutubeTranscriptError extends Error {
  constructor(message: string) {
    super(`[YoutubeTranscript] ${message}`);
  }
}

type YtFetchConfig = {
  lang?: string; // Object with lang param (eg: en, es, hk, uk) format.
};

async function fetchTranscript(videoId: string, config: YtFetchConfig = {}) {
  console.log("fetchTranscript", videoId);
  const identifier = extractYouTubeID(videoId);
  const lang = config?.lang ?? "en";
  try {
    const transcriptUrl = await fetch(
      `https://www.youtube.com/watch?v=${identifier}`,
      {
        headers: {
          "User-Agent": USER_AGENT,
        },
      }
    )
      .then((res) => res.text())
      .then((html) => parse(html))
      .then((html) => parseTranscriptEndpoint(html, lang));

    if (!transcriptUrl)
      throw new Error("Failed to locate a transcript for this video!");

    // Result is hopefully some XML.
    const transcriptXML = await fetch(transcriptUrl)
      .then((res) => res.text())
      .then((xml) => parse(xml));

    const chunks = transcriptXML.getElementsByTagName("text");

    let transcriptions = [];
    for (const chunk of chunks) {
      const [offset, duration] = chunk.rawAttrs.split(" ");
      transcriptions.push({
        text: chunk.text,
        offset: convertToMs(offset),
        duration: convertToMs(duration),
      });
    }
    return transcriptions;
  } catch (e: any) {
    throw new YoutubeTranscriptError(e);
  }
}

function convertToMs(text: string) {
  const float = parseFloat(text.split("=")[1].replace(/"/g, "")) * 1000;
  return Math.round(float);
}

function parseTranscriptEndpoint(document: any, langCode?: string) {
  try {
    // Get all script tags on document page
    const scripts = document.getElementsByTagName("script");

    // find the player data script.
    const playerScript = scripts.find((script: any) =>
      script.textContent.includes("var ytInitialPlayerResponse = {")
    );

    const dataString =
      playerScript.textContent
        ?.split("var ytInitialPlayerResponse = ")?.[1] //get the start of the object {....
        ?.split("};")?.[0] + // chunk off any code after object closure.
      "}"; // add back that curly brace we just cut.

    const data = JSON.parse(dataString.trim()); // Attempt a JSON parse
    const availableCaptions =
      data?.captions?.playerCaptionsTracklistRenderer?.captionTracks || [];

    // If languageCode was specified then search for it's code, otherwise get the first.
    let captionTrack = availableCaptions?.[0];
    if (langCode)
      captionTrack =
        availableCaptions.find((track: any) =>
          track.languageCode.includes(langCode)
        ) ?? availableCaptions?.[0];

    return captionTrack?.baseUrl;
  } catch (e: any) {
    console.error(`parseTranscriptEndpoint Error: ${e.message}`);
    return null;
  }
}

export function extractYouTubeID(urlOrID: string): string | null {
  // Regular expression for YouTube ID format
  const regExpID = /^[a-zA-Z0-9_-]{11}$/;

  // Check if the input is a YouTube ID
  if (regExpID.test(urlOrID)) {
    return urlOrID;
  }

  // Regular expression for standard YouTube links
  const regExpStandard = /youtube\.com\/watch\?v=([a-zA-Z0-9_-]+)/;

  // Regular expression for YouTube Shorts links
  const regExpShorts = /youtube\.com\/shorts\/([a-zA-Z0-9_-]+)/;

  // Check for standard YouTube link
  const matchStandard = urlOrID.match(regExpStandard);

  if (matchStandard) {
    return matchStandard[1];
  }

  // Check for YouTube Shorts link
  const matchShorts = urlOrID.match(regExpShorts);
  if (matchShorts) {
    return matchShorts[1];
  }

  // Return null if no match is found
  return null;
}

export { fetchTranscript, YoutubeTranscriptError };
```

You will need to install the `node-html-parser` dependency.

```bash
yarn add node-html-parser
```

Once everything is installed, let's implement this in our route handles and see if we can get your YouTube transcript.

Navigate to `src/app/api/summarize/route.ts` and let's import our `fetchTranscript` function that we just created from `src/lib/youtube-transcript.ts`.

We will also create a wrapper function that will call this service. Here is what it will look like.

Import the `fetchTranscript` function we just added.

```ts
import { fetchTranscript } from "@/lib/youtube-transcript";
```

Let's make the following changes.

```ts
const body = await req.json();
const videoId = body.videoId;

let transcript: Awaited<ReturnType<typeof fetchTranscript>>;

try {
  transcript = await fetchTranscript(videoId);
} catch (error) {
  console.error("Error processing request:", error);
  if (error instanceof Error)
    return new Response(JSON.stringify({ error: error.message }));
  return new Response(JSON.stringify({ error: "Error getting transcript." }));
}

console.log("Transcript:", transcript);
```

We will get our `videoId` from the `req` and pass it into our `getTranscript` function.

The completed code should look like the following.

```ts
import { NextRequest } from "next/server";
import { fetchTranscript } from "@/lib/youtube-transcript";

async function getTranscript(id: string) {
  try {
    return await fetchTranscript(id);
  } catch (error) {
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(
      JSON.stringify({ error: "Failed to generate summary." })
    );
  }
}

export async function POST(req: NextRequest) {
  console.log("FROM OUR ROUTE HANDLER:", req.body);
  const body = await req.json();
  const videoId = body.videoId;

  let transcript: Awaited<ReturnType<typeof getTranscript>>;

  try {
    transcript = await getTranscript(videoId);
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }

  console.log("Transcript:", transcript);

  try {
    return new Response(
      JSON.stringify({ data: "return from our handler", error: null }),
      {
        status: 200,
      }
    );
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }
}
```

Now, let's test our front end.

![](./images/004-test-transcript.gif)

Once we submit our form, we should see the following output in our console.

```js
Transcript: [
  {
    text: "In the last chapter, you and I started to step ",
    offset: 0,
    duration: 2009,
  },
  {
    text: "through the internal workings of a transformer.",
    offset: 2009,
    duration: 2010,
  },
  {
    text: "This is one of the key pieces of technology inside large language models, ",
    offset: 4560,
    duration: 3365,
  },
  // rest of the items
];
```

Nice. We are getting our transcript, but it is an array that includes the time code. We want the text. Let's create a utility function to do this conversion for us.

Let's add this function to the top of our code in our route handler.

```ts
function transformData(data: any[]) {
  let text = "";

  data.forEach((item) => {
    text += item.text + " ";
  });

  return {
    data: data,
    text: text.trim(),
  };
}
```

Now, let's use it.

```ts
const transformedData = transformData(transcript);
console.log("Transcript:", transformedData);
```

The updated code should look like the following.

```ts
import { NextRequest } from "next/server";
import { fetchTranscript } from "@/lib/youtube-transcript";

async function getTranscript(id: string) {
  try {
    return await fetchTranscript(id);
  } catch (error) {
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(
      JSON.stringify({ error: "Failed to generate summary." })
    );
  }
}

function transformData(data: any[]) {
  let text = "";

  data.forEach((item) => {
    text += item.text + " ";
  });

  return {
    data: data,
    text: text.trim(),
  };
}

export async function POST(req: NextRequest) {
  console.log("FROM OUR ROUTE HANDLER:", req.body);
  const body = await req.json();
  const videoId = body.videoId;

  let transcript: Awaited<ReturnType<typeof getTranscript>>;

  try {
    transcript = await getTranscript(videoId);
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }

  const transformedData = transformData(transcript);
  console.log("Transcript:", transformedData);

  try {
    return new Response(
      JSON.stringify({ data: "return from our handler", error: null }),
      {
        status: 200,
      }
    );
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }
}
```

Let's test it out. We should now see a response containing just our text.

```bash
  text: 'In the last chapter, you and I started to step through the internal workings of a transformer. This is one of the key pieces of technology inside large language models,  and a lot of other tools in the modern wave of AI. It first hit the scene in a now-famous 2017 paper called Attention is All You Need,  and in this chapter you and I will dig '... 18362 more characters
```

Excellent. Now that we have our transcript, we can use it to prepare our summary.

But first, let's add some basic validation.

Let's check our form to ensure we provide a proper video ID or URL.

Then, we will check if our user is logged in and has credits.

### Form Validation and Submission

First, let's add a util function to extract a valid videoId from our YouTube url. We will use it to validate the YouTube video ID and our.

And the following is inside our `utils.ts` file.

```ts
export function extractYouTubeID(urlOrID: string): string | null {
  // Regular expression for YouTube ID format
  const regExpID = /^[a-zA-Z0-9_-]{11}$/;

  // Check if the input is a YouTube ID
  if (regExpID.test(urlOrID)) {
    return urlOrID;
  }

  // Regular expression for standard YouTube links
  const regExpStandard = /youtube\.com\/watch\?v=([a-zA-Z0-9_-]+)/;

  // Regular expression for YouTube Shorts links
  const regExpShorts = /youtube\.com\/shorts\/([a-zA-Z0-9_-]+)/;

  // Check for standard YouTube link
  const matchStandard = urlOrID.match(regExpStandard);
  if (matchStandard) {
    return matchStandard[1];
  }

  // Check for YouTube Shorts link
  const matchShorts = urlOrID.match(regExpShorts);
  if (matchShorts) {
    return matchShorts[1];
  }

  // Return null if no match is found
  return null;
}
```

Let's navigate to our `SummaryForm.tsx` file and import our `extractYouTubeID`.

```tsx
import { extractYouTubeID } from "@/lib/utils";
```

And update it with the following changes inside our `handleFormSubmit` function.

```tsx
async function handleFormSubmit(event: React.FormEvent<HTMLFormElement>) {
  event.preventDefault();
  setLoading(true);
  toast.success("Submitting Form");

  const formData = new FormData(event.currentTarget);
  const videoId = formData.get("videoId") as string;

  const processedVideoId = extractYouTubeID(videoId);

  if (!processedVideoId) {
    toast.error("Invalid Youtube Video ID");
    setLoading(false);
    setValue("");
    setError({
      ...INITIAL_STATE,
      message: "Invalid Youtube Video ID",
      name: "Invalid Id",
    });
    return;
  }

  toast.success("Generating Summary");

  const summaryResponseData = await generateSummaryService(videoId);
  console.log(summaryResponseData, "Response from route handler");

  toast.success("Summary Created");
  setLoading(false);
}
```

Let's test our front end.

![](./images/005-test-error.gif)

Nice, we can check if we have the proper Url or ID.

Now, let's navigate to our route handler at `src/app/api/summarize/route.ts` and add a check to check if a user is logged in and has available credits.

First, import the following helper methods.

```ts
import { getUserMeLoader } from "@/data/services/get-user-me-loader";
import { getAuthToken } from "@/data/services/get-token";
```

And the following lines are inside the `POST` function.

```ts
export async function POST(req: NextRequest) {
  const user = await getUserMeLoader();
  const token = await getAuthToken();

  if (!user.ok || !token)
    return new Response(
      JSON.stringify({ data: null, error: "Not authenticated" }),
      { status: 401 }
    );

  if (user.data.credits < 1)
    return new Response(
      JSON.stringify({
        data: null,
        error: "Insufficient credits",
      }),
      { status: 402 }
    );

  // rest of code
}
```

The final code should look like the following.

```ts
import { NextRequest } from "next/server";
import { fetchTranscript } from "@/lib/youtube-transcript";

import { getUserMeLoader } from "@/data/services/get-user-me-loader";
import { getAuthToken } from "@/data/services/get-token";

async function getTranscript(id: string) {
  try {
    return await fetchTranscript(id);
  } catch (error) {
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(
      JSON.stringify({ error: "Failed to generate summary." })
    );
  }
}

function transformData(data: any[]) {
  let text = "";

  data.forEach((item) => {
    text += item.text + " ";
  });

  return {
    data: data,
    text: text.trim(),
  };
}

export async function POST(req: NextRequest) {
  const user = await getUserMeLoader();
  const token = await getAuthToken();

  if (!user.ok || !token)
    return new Response(
      JSON.stringify({ data: null, error: "Not authenticated" }),
      { status: 401 }
    );

  if (user.data.credits < 1)
    return new Response(
      JSON.stringify({
        data: null,
        error: "Insufficient credits",
      }),
      { status: 402 }
    );

  console.log("FROM OUR ROUTE HANDLER:", req.body);
  const body = await req.json();
  const videoId = body.videoId;

  let transcript: Awaited<ReturnType<typeof getTranscript>>;

  try {
    transcript = await getTranscript(videoId);
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }

  const transformedData = transformData(transcript);
  console.log("Transcript:", transformedData);

  try {
    return new Response(
      JSON.stringify({ data: "return from our handler", error: null }),
      {
        status: 200,
      }
    );
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }
}
```

We must add a check in our `SummaryForm.tsx` file to handle the errors inside our `handleFormSubmit` function.

Let's add the following code after this line.

```tsx
const summaryResponseData = await generateSummaryService(videoId);
console.log(summaryResponseData, "Response from route handler");

// add the following

if (summaryResponseData.error) {
  setValue("");
  toast.error(summaryResponseData.error);
  setError({
    ...INITIAL_STATE,
    message: summaryResponseData.error,
    name: "Summary Error",
  });
  setLoading(false);
  return;
}

// rest of the code
```

The completed code should look like the following.

```tsx
async function handleFormSubmit(event: React.FormEvent<HTMLFormElement>) {
  event.preventDefault();
  setLoading(true);
  toast.success("Submitting Form");

  const formData = new FormData(event.currentTarget);
  const videoId = formData.get("videoId") as string;

  const processedVideoId = extractYouTubeID(videoId);

  if (!processedVideoId) {
    toast.error("Invalid Youtube Video ID");
    setLoading(false);
    setValue("");
    setError({
      ...INITIAL_STATE,
      message: "Invalid Youtube Video ID",
      name: "Invalid Id",
    });
    return;
  }

  toast.success("Generating Summary");

  const summaryResponseData = await generateSummaryService(videoId);
  console.log(summaryResponseData, "Response from route handler");

  if (summaryResponseData.error) {
    setValue("");
    toast.error(summaryResponseData.error);
    setError({
      ...INITIAL_STATE,
      message: summaryResponseData.error,
      name: "Summary Error",
    });
    setLoading(false);
    return;
  }

  toast.success("Summary Created");
  setLoading(false);
}
```

Now, let's test our form.

![006-testing-credits](./images/006-testing-credits.gif)

Excellent, it is working. Now, we are ready to implement our logic to get our summary. Let's do it.

### Generate Summary with LangChain and OpenAI in Next.js

Now, let's write our logic to handle the generation of our summary with OpenAi and LangChain.

If you never used LangChain before, it is a tool that helps you simplify building AI-powered apps. You can learn about it [here](https://js.langchain.com/docs/get_started/introduction).

![007-langchain](./images/007-langchain.png)

Before starting, install the following packages `@langchain/openai` and `langchain` with the following command.

```bash
yarn add @langchain/openai langchain
```

Nice. Now, let's make the following changes in our route handler: Navigate to `src/app/api/summarize/route.ts` and make the following changes.

First, let's import all the required dependencies.

```ts
import { ChatOpenAI } from "@langchain/openai";
import { PromptTemplate } from "@langchain/core/prompts";
import { StringOutputParser } from "@langchain/core/output_parsers";
```

Next, create a function called `generateSummary` with the following code.

```ts
async function generateSummary(content: string, template: string) {
  const prompt = PromptTemplate.fromTemplate(template);

  const model = new ChatOpenAI({
    openAIApiKey: process.env.OPENAI_API_KEY,
    modelName: process.env.OPENAI_MODEL ?? "gpt-4-turbo-preview",
    temperature: process.env.OPENAI_TEMPERATURE
      ? parseFloat(process.env.OPENAI_TEMPERATURE)
      : 0.7,
    maxTokens: process.env.OPENAI_MAX_TOKENS
      ? parseInt(process.env.OPENAI_MAX_TOKENS)
      : 4000,
  });

  const outputParser = new StringOutputParser();
  const chain = prompt.pipe(model).pipe(outputParser);

  try {
    const summary = await chain.invoke({ text: content });
    return summary;
  } catch (error) {
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(
      JSON.stringify({ error: "Failed to generate summary." })
    );
  }
}
```

Now, let's create a prompt template.

```ts
const TEMPLATE = `
INSTRUCTIONS: 
  For the this {text} complete the following steps.
  Generate the title for based on the content provided
  Summarize the following content and include 5 key topics, writing in first person using normal tone of voice.
  
  Write a youtube video description
    - Include heading and sections.  
    - Incorporate keywords and key takeaways

  Generate bulleted list of key points and benefits

  Return possible and best recommended key words
`;
```

You can change the template accordingly to suit your needs.

Let's use the `generateSummary` function inside our `POST` function. Update the code with the following.

The updated `POST` function should look like the following.

```ts
export async function POST(req: NextRequest) {
  const user = await getUserMeLoader();
  const token = await getAuthToken();

  if (!user.ok || !token)
    return new Response(
      JSON.stringify({ data: null, error: "Not authenticated" }),
      { status: 401 }
    );

  if (user.data.credits < 1)
    return new Response(
      JSON.stringify({
        data: null,
        error: "Insufficient credits",
      }),
      { status: 402 }
    );

  console.log("FROM OUR ROUTE HANDLER:", req.body);
  const body = await req.json();
  const videoId = body.videoId;

  let transcript: Awaited<ReturnType<typeof fetchTranscript>>;

  try {
    transcript = await fetchTranscript(videoId);
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(JSON.stringify({ error: "Error getting transcript." }));
  }

  const transformedData = transformData(transcript);
  console.log("Transformed Data", transformedData);

  let summary: Awaited<ReturnType<typeof generateSummary>>;

  try {
    summary = await generateSummary(transformedData.text, TEMPLATE);
    return new Response(JSON.stringify({ data: summary, error: null }));
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error.message }));
    return new Response(JSON.stringify({ error: "Error generating summary." }));
  }
}
```

In the code above, we implemented our `generateSummary` function, which will generate our summary and send it back to you via our form. We will also create a server action responsible for saving our data into our Strapi backend.

But first, let's console.log the output to see if we are getting back our summary.

First, create a `.env.local` file and add our Open AI API key.

**WARNING**: Ensure your `.gitignore` file ignores the `.env*.local` file from your commit so you don't leak your Open AI key.

```
# local env files
.env*.local
```

```bash
OPENAI_API_KEY=ADD_YOUR_KEY_HERE
```

Let's test our form and see if we can get our summary. Make sure to add some credits to your user.

Excellent. We can see our output on the console.

```markdown
**Title:** Quickstart Guide to Launching Your Project with Strapi in Just 3 Minutes

**YouTube Video Description:**

**Heading:** Fast Track Your Development with Strapi: A 3-Minute Quickstart Guide

**Introduction:**
Join me today as we explore how to get your project up and running with Strapi in just three minutes! Strapi is an open-source headless CMS that simplifies the process of building, managing, and deploying content. Whether you're a developer, content creator, or project manager, this guide is designed to help you kickstart your project effortlessly.

**Sections:**

- **Setting Up Your Strapi Project:**
  - Learn how to create a new Strapi project using the quickstart command to leverage default configurations, including setting up an SQLite database.

Rest of summary...
```

Now that we know we are getting our summary, the last step is to save it to Strapi and deduct 1 credit.

### Saving Our Summary To Strapi

First, create a new `collection-type` in Strapi admin to save our summary.

Navigate to the content builder page and create a new collection named `Summary` with the following fields.

![008-create-summary](./images/008-create-summary.png)

Let's add the following fields.

| Name    | Field     | Type        |
| ------- | --------- | ----------- |
| videoId | Text      | Short Text  |
| title   | Text      | Short Text  |
| summary | Rich Text | Markdown    |
| user    | Relation  | One to many |

When creating user relations, make sure you select the appropriate relation.

![](./images/009-user-relation.png)

Here is what the final fields will look like.

![](./images/010-summary-fields.png)

Now, navigate to `Setting` and add the following permissions.

![](./images/011-set-permissions.png)

Now that we have our **Summary** `collection-type`, let's create a server action to save our data to Strapi.

Let's start by navigating to `srs/data/actions`, creating a new file called `summary-actions.ts`, and adding the following code.

```ts
"use server";

import { getAuthToken } from "@/data/services/get-token";
import { mutateData } from "@/data/services/mutate-data";
import { flattenAttributes } from "@/lib/utils";
import { redirect } from "next/navigation";

interface Payload {
  data: {
    title?: string;
    videoId: string;
    summary: string;
  };
}

export async function createSummaryAction(payload: Payload) {
  const authToken = await getAuthToken();
  if (!authToken) throw new Error("No auth token found");

  const data = await mutateData("POST", "/api/summaries", payload);
  const flattenedData = flattenAttributes(data);
  redirect("/dashboard/summaries/" + flattenedData.id);
}
```

Now that we have our `createSummaryAction`, let's use it in our `handleFormSubmit,` found in our form named `SummaryForm`.

First, let's import our newly created action.

```tsx
import { createSummaryAction } from "@/data/actions/summary-actions";
```

Update the `handleFormSubmit` with the following code.

```tsx
async function handleFormSubmit(event: React.FormEvent<HTMLFormElement>) {
  event.preventDefault();
  setLoading(true);
  toast.success("Submitting Form");

  const formData = new FormData(event.currentTarget);
  const videoId = formData.get("videoId") as string;

  const processedVideoId = extractYouTubeID(videoId);

  if (!processedVideoId) {
    toast.error("Invalid Youtube Video ID");
    setLoading(false);
    setValue("");
    setError({
      ...INITIAL_STATE,
      message: "Invalid Youtube Video ID",
      name: "Invalid Id",
    });
    return;
  }

  toast.success("Generating Summary");

  const summaryResponseData = await generateSummaryService(videoId);
  console.log(summaryResponseData, "Response from route handler");

  if (summaryResponseData.error) {
    setValue("");
    toast.error(summaryResponseData.error);
    setError({
      ...INITIAL_STATE,
      message: summaryResponseData.error,
      name: "Summary Error",
    });
    setLoading(false);
    return;
  }

  const payload = {
    data: {
      title: `Summary for video: ${processedVideoId}`,
      videoId: processedVideoId,
      summary: summaryResponseData.data,
    },
  };

  try {
    await createSummaryAction(payload);
  } catch (error) {
    toast.error("Error Creating Summary");
    setError({
      ...INITIAL_STATE,
      message: "Error Creating Summary",
      name: "Summary Error",
    });
    setLoading(false);
    return;
  }

  toast.success("Summary Created");
  setLoading(false);
}
```

The above code will be responsible for saving our data into Strapi.

Let's do a quick test and see if it works. We should be redirected to our `summaries` route, which we have yet to create, so we will get our not found page. This is okay, and we will fix it soon.

![](./images/013-redirect.gif)

But you should see your data in your Strapi Admin.

![](./images/013-1-response.png)

You will notice that we are not setting our user or deducting one credit on creation. We will do this in Strapi by creating custom middleware. But first, let's finish all of our Next.js UI.

## Create Summary Page Card View

Let's navigate to our `dashboard` folder. Inside, create another folder named `summaries` with a `page.tsx` file and paste it into the following code.

```tsx
import Link from "next/link";
import { getSummaries } from "@/data/loaders";

import { Card, CardContent, CardHeader, CardTitle } from "@/components/ui/card";

interface LinkCardProps {
  id: string;
  title: string;
  summary: string;
}

function LinkCard({ id, title, summary }: Readonly<LinkCardProps>) {
  return (
    <Link href={`/dashboard/summaries/${id}`}>
      <Card className="relative">
        <CardHeader>
          <CardTitle className="leading-8 text-pink-500">
            {title || "Video Summary"}
          </CardTitle>
        </CardHeader>
        <CardContent>
          <p className="w-full mb-4 leading-7">
            {summary.slice(0, 164) + " [read more]"}
          </p>
        </CardContent>
      </Card>
    </Link>
  );
}

export default async function SummariesRoute() {
  const { data } = await getSummaries();
  if (!data) return null;
  return (
    <div className="grid grid-cols-1 gap-4 p-4">
      <div className="grid grid-cols-1 sm:grid-cols-2 md:grid-cols-3 lg:grid-cols-4 gap-4">
        {data.map((item: LinkCardProps) => (
          <LinkCard key={item.id} {...item} />
        ))}
      </div>
    </div>
  );
}
```

Before this component works, we must create the `getSummaries` function to load our data.

Let's navigate to our `loaders.ts` file and make the following changes.

First, import the following method to give us access to our auth token for authorized requests.

```ts
import { getAuthToken } from "./services/get-token";
```

Next, update the `fetchData` function with the following code.

```ts
async function fetchData(url: string) {
  const authToken = await getAuthToken();

  const headers = {
    method: "GET",
    headers: {
      "Content-Type": "application/json",
      Authorization: `Bearer ${authToken}`,
    },
  };

  try {
    const response = await fetch(url, authToken ? headers : {});
    const data = await response.json();
    return flattenAttributes(data);
  } catch (error) {
    console.error("Error fetching data:", error);
    throw error;
  }
}
```

Now, let's add this code at the end of the file to get our summaries.

```ts
export async function getSummaries() {
  const url = new URL("/api/summaries", baseUrl);
  return fetchData(url.href);
}
```

Now, we need to update the following permissions in the Strapi dashboard.

![](./images/014-permissions.gif)

Under `Settings` => `User Permissions` => `Roles` => `Authenticated`, set the **Global** and **Home-page** checkboxes to `checked`.

Restart your application, and you should now be able to view the list.

![](./images/014-1-list.png)

Now, let's create the Single Card view.

First, we will create a dynamic route; you can learn more about dynamic routes in Next.js docs [here](https://nextjs.org/docs/pages/building-your-application/routing/dynamic-routes).

"Dynamic Routes are pages that allow you to add custom params to your URLs."

Create a new folder called `[videoId]` inside the' summaries' folder.

Inside our newly created dynamic route, create a file named `layout.tsx` with the following code.

```tsx
import { extractYouTubeID } from "@/lib/utils";

export default async function SummarySingleRoute({
  params,
  children,
}: {
  readonly params: any;
  readonly children: React.ReactNode;
}) {
  return (
    <div>
      <div className="h-full grid gap-4 grid-cols-5 p-4">
        <div className="col-span-3">{children}</div>
        <div className="col-span-2">
          <div>
            <p>Video will go here</p>
          </div>
        </div>
      </div>
    </div>
  );
}
```

Create another file called `page.tsx` file with the following.

```tsx
interface ParamsProps {
  params: {
    videoId: string;
  };
}

export default async function SummaryCardRoute({
  params,
}: Readonly<ParamsProps>) {
  return <p>Summary card with go here: {params.videoId}</p>;
}
```

Now that we know our pages work, let's create the loaders to get the appropriate data.

### Fetching And Displaying Our Single Video and Summary

Let's start by navigating our `loaders.ts` file and adding the following functions.

```tsx
export async function getSummaryById(summaryId: string) {
  return fetchData(`${baseUrl}/api/summaries/${summaryId}`);
}
```

Now, before using our `getSummaryById` function, let's install our video player. We will use **React Player** that you can find [here](https://www.npmjs.com/package/react-player).

Let's start by installing it with the following command.

```bash
yarn add react-player
```

Let's create a wrapper component using our **React Player** inside the `components/custom` folder. Create a `YouTubePlayer.tsx` file and paste it into the following code.

```tsx
"use client";
import ReactPlayer from "react-player/youtube";

function generateYouTubeUrl(videoId: string) {
  const baseUrl = new URL("https://www.youtube.com/watch");
  baseUrl.searchParams.append("v", videoId);
  return baseUrl.href;
}

interface YouTubePlayerProps {
  videoId: string | null;
}

export default function YouTubePlayer({
  videoId,
}: Readonly<YouTubePlayerProps>) {
  if (!videoId) return null;
  const videoUrl = generateYouTubeUrl(videoId);

  return (
    <div className="relative aspect-video rounded-md overflow-hidden">
      <ReactPlayer
        url={videoUrl}
        width="100%"
        height="100%"
        controls
        className="absolute top-0 left-0"
      />
    </div>
  );
}
```

Now that we have our react player let's update the `layout.tsx` file using the following code.

```tsx
import dynamic from "next/dynamic";

import { extractYouTubeID } from "@/lib/utils";
import { getSummaryById } from "@/data/loaders";

const NoSSR = dynamic(() => import("@/components/custom/YouTubePlayer"), {
  ssr: false,
});

export default async function SummarySingleRoute({
  params,
  children,
}: {
  readonly params: any;
  readonly children: React.ReactNode;
}) {
  const data = await getSummaryById(params.videoId);
  const videoId = extractYouTubeID(data.videoId);
  if (!data) return <p>No Items Found</p>;
  return (
    <div>
      <div className="h-full grid gap-4 grid-cols-5 p-4">
        <div className="col-span-3">{children}</div>
        <div className="col-span-2">
          <div>
            <NoSSR videoId={videoId} />
          </div>
        </div>
      </div>
    </div>
  );
}
```

In the code above, we use `dynamic` to disable SSR, which helps avoid issues when using some client-side components. In this blog post, you can read more [here](https://dev.to/kawanedres/leveraging-nextjs-dynamic-imports-to-solve-hydration-problems-5086).

Next.js docs reference on solving hydration issues [here](https://nextjs.org/docs/messages/react-hydration-error#solution-2-disabling-ssr-on-specific-components)

![](./images/015-video.png)

Now, let's display our summary.

Let's first create a new file called `SummaryCardForm.tsx`. We can add it to our `src/components/forms` folder and paste it into the following code.

```tsx
// import { updateSummaryAction, deleteSummaryAction } from "@/data/actions/summary-actions";

import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { cn } from "@/lib/utils";

import {
  Card,
  CardContent,
  CardFooter,
  CardHeader,
  CardTitle,
} from "@/components/ui/card";

import { SubmitButton } from "@/components/custom/SubmitButton";
import { DeleteButton } from "@/components/custom/DeleteButton";

export function SummaryCardForm({
  item,
  className,
}: {
  readonly item: any;
  readonly className?: string;
}) {
  // const deleteSummaryById = deleteSummaryAction.bind(null, item.id);

  return (
    <Card className={cn("mb-8 relative h-auto", className)}>
      <CardHeader>
        <CardTitle>Video Summary</CardTitle>
      </CardHeader>
      <CardContent>
        <div>
          <form>
            <Input
              id="title"
              name="title"
              placeholder="Update your title"
              required
              className="mb-4"
              defaultValue={item.title}
            />
            <Textarea
              name="summary"
              className="flex w-full rounded-md bg-transparent px-3 py-2 text-sm shadow-sm placeholder:text-muted-foreground focus-visible:outline-none focus-visible:bg-gray-50 focus-visible:ring-1 focus-visible:ring-ring disabled:cursor-not-allowed disabled:opacity-50 mb-4 h-[calc(100vh-245px)] "
              defaultValue={item.summary}
            />
            <input type="hidden" name="id" value={item.id} />
            <SubmitButton
              text="Update Summary"
              loadingText="Updating Summary"
            />
          </form>
          <form>
            <DeleteButton className="absolute right-4 top-4 bg-red-700 hover:bg-red-600" />
          </form>
        </div>
      </CardContent>
      <CardFooter></CardFooter>
    </Card>
  );
}
```

We are using a new component, **DeleteButton**. Let's create it inside our `components/custom` folder. Create a `DeleteButton.tsx` file and add the following code.

```tsx
"use client";
import { useFormStatus } from "react-dom";
import { cn } from "@/lib/utils";
import { Button } from "@/components/ui/button";
import { TrashIcon } from "lucide-react";
import { Loader2 } from "lucide-react";

function Loader() {
  return (
    <div className="flex items-center">
      <Loader2 className="h-4 w-4 animate-spin" />
    </div>
  );
}

interface DeleteButtonProps {
  className?: string;
}

export function DeleteButton({ className }: Readonly<DeleteButtonProps>) {
  const status = useFormStatus();
  return (
    <Button
      type="submit"
      aria-disabled={status.pending}
      disabled={status.pending}
      className={cn(className)}
    >
      {status.pending ? <Loader /> : <TrashIcon className="w-4 h-4" />}
    </Button>
  );
}
```

Let's update our `page.tsx` file with the following code.

```tsx
import { getSummaryById } from "@/data/loaders";
import { SummaryCardForm } from "@/components/forms/SummaryCardForm";

interface ParamsProps {
  params: {
    videoId: string;
  };
}

export default async function SummaryCardRoute({
  params,
}: Readonly<ParamsProps>) {
  const data = await getSummaryById(params.videoId);
  if (!data) return <p>No Items Found</p>;
  return <SummaryCardForm item={data} />;
}
```

![](./images/016-summary.png)

![](./images/017-generating-summary.gif)

Now that we know our front end works. Let's revisit our route handler.

## Using Strapi Route Middleware To Set User/Summary Relation

When creating our summary, we will set the summary/user relation on the backend, where we can confirm the logged-in user creating the content.

This will prevent anyone from the front end from passing a user ID that is not their own.

We are also not handling user credit updates; let's do that in the middleware.

**What is a route middleware**

In Strapi, a route middleware is a type of middleware that has a more limited scope compared to global middlewares.

They control the request flow and can change the request itself before moving forward.

They can also be used to control access to a route and perform additional logic.

For example, they can be used instead of policies to control access to an endpoint. They could modify the context before passing it down to further core elements of the Strapi server.

You can learn more about route middlewares [here](https://docs.strapi.io/dev-docs/backend-customization/middlewares).

Let's first start by creating our route middleware.

We can use our cli command. In your `backend` folder, run the following command.

```bash
yarn strapi generate
```

Choose to generate a middleware option.

```bash

▶ yarn strapi generate
yarn strapi generate
yarn run v1.22.19
$ strapi generate
? Strapi Generators
  content-type - Generate a content type for an API
  plugin - Generate a basic plugin
  policy - Generate a policy for an API
❯ middleware - Generate a middleware for an API
  migration - Generate a migration
  service - Generate a service for an API
  api - Generate a basic API
(Move up and down to reveal more choices)
```

Let's call it `on-summary-create` and add it to an existing API. Which will be `summary`

```bash
? Strapi Generators middleware - Generate a middleware for an API
? Middleware name on-summary-create
? Where do you want to add this middleware?
  Add middleware to root of project
❯ Add middleware to an existing API
  Add middleware to an existing plugin
```

```bash
? Which API is this for?
  global
  home-page
❯ summary

```

Now, let's take a look in the following folder: `backend/src/api/summary/middlewares.` You should see the following file: `on-summary-create` with the following boilerplate.

```js
"use strict";

/**
 * `on-summary-create` middleware
 */

module.exports = (config, { strapi }) => {
  // Add your own logic here.
  return async (ctx, next) => {
    strapi.log.info("In on-summary-create middleware.");

    await next();
  };
};
```

Let's update it with the following code.

```js
"use strict";

module.exports = (config, { strapi }) => {
  return async (ctx, next) => {
    const user = ctx.state.user;
    if (!user) return ctx.unauthorized("You are not authenticated");

    const availableCredits = user.credits;

    if (availableCredits === 0)
      return ctx.unauthorized("You do not have enough credits.");

    await next();

    // update the user's credits
    const uid = "plugin::users-permissions.user";
    const payload = {
      data: {
        credits: availableCredits - 1,
        summaries: {
          connect: [ctx.response.body.data.id],
        },
      },
    };

    try {
      await strapi.entityService.update(uid, user.id, payload);
    } catch (error) {
      ctx.badRequest("Error Updating User Credits");
    }

    console.log("############ Inside middleware end #############");
  };
};
```

In the code above, we deduct one credit and set the user and summary relation.

Before testing it out, we have to enable it inside our route.

You can learn more about Strapi's routes [here](https://docs.strapi.io/dev-docs/backend-customization/routes).

Navigate to the `backend/src/api/summary/routes/summary.js` file and update with the following.

```js
"use strict";

/**
 * summary router
 */

const { createCoreRouter } = require("@strapi/strapi").factories;

module.exports = createCoreRouter("api::summary.summary", {
  config: {
    create: {
      middlewares: ["api::summary.on-summary-create"],
    },
  },
});
```

Now, our middleware will fire when we create a new summary.

Now, restart your Strapi backend and Next.js frontend and create a new summary.

You will see that we are now setting our user data.

![](./images/020-setting-user.png)

## Conclusion

In part 6 of our Next.js 14 tutorial series, we tackled generating video summaries using Open AI and LangChain, a highlight feature for our Next.js app.

We built a SummaryForm component to handle user submissions and explored Next.js API routes for server-side logic.

We then leveraged OpenAI to summarize video transcripts, demonstrating the practical use of AI in web development.

In the next post, we will examine our summary details page and discuss updating and deleting posts.

As well as how to add policies to ensure that our user can only modify their content.

Hope you are enjoying this series as much as I am making it.

If you have any questions or have found "bugs," let us know so we can continue to improve our content.
