# Epic Next JS 14 TutorialPart 4: Create Video Summary with Next.js, LangChain, and Open AI.

In the previous tutorial, we completed our **Dashboard** and **Account** pages, in this section we will work on setting up our **Summaries** form and **Summary** page to show our recent summaries, as well as detailed page to show our summary and video.

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

We are going to kick of the tutorial by starting to work on our **SummaryForm** component. This time around, instead of using a `server action` we will explore how to create and **api** route in Next.js.

You can learn more about Next.js route handlers [here](https://nextjs.org/docs/app/building-your-application/routing/route-handlers)

But first, let's start by creating our **Summary Form** that we can use to submit the request.

Navigate to `src/components/forms` and create a new file called `SummaryForm.tsx` and paste in the following code as the starting point.

```jsx
"use client";

import { useState } from "react";
import { toast } from "sonner";
import { cn } from "@/lib/utils";

import { Input } from "@/components/ui/input";
import { SubmitButton } from "@/components/custom/SubmitButton";

interface StrapiErrorsProps {
  message: string | null;
  name: string;
  status: string | null;
}

const INITIAL_STATE = {
  message: null,
  name: "",
  status: null,
};

export function SummaryForm() {
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState < StrapiErrorsProps > INITIAL_STATE;
  const [value, setValue] = useState < string > "";

  async function handleFormSubmit(event: React.FormEvent<HTMLFormElement>) {
    event.preventDefault();
    setLoading(true);
    toast.success("Testing Toast");
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

The above code contains a basic form UI, and a `handleFormSubmit` function which has not been implemented yet.

We are also using **Sonner** one of my favorite toast libraries. You can learn more about it [here](https://sonner.emilkowal.ski/).

But we are not using it directly, and instead using the **Chadcn UI** component that you can find [here](https://ui.shadcn.com/docs/components/sonner).

```bash
npx shadcn-ui@latest add sonner
```

Once **Sonner** is installed, let's implemented in our main `layout.tsx` file by adding the following import.

```jsx
import { Toaster } from "@/components/ui/sonner";
```

And adding the code below above our `TopNav`.

```jsx
<body className={inter.className}>
  <Toaster position="bottom-center" />
  <Header data={globalData.header} />
  <div>{children}</div>
  <Footer data={globalData.footer} />
</body>
```

Now, let's add this form to our top navigation by navigating to `src/components/custom/Header.tsx` file and making the following changes.

```jsx
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

Now let's restart our frontend project and see if it is showing up.

![](./images/002-toast-and-form.gif)

Now that we have our basic form working. Let's take a look how we can set up our first API Handler Route in Next.js

## How To Create A Route Handler in Next.js

We will have the Next.js [docs](https://nextjs.org/docs/app/building-your-application/routing/route-handlers) open as reference.

Let's start by creating a new folder inside our `app` directory called `api` with a folder called `summarize` with a file called `route.ts` and paste in the following code.

```ts
import { NextRequest } from "next/server";

export async function POST(req: NextRequest) {
  console.log("FROM OUR ROUTE HANDLER:", req.body);
  try {
    return new Response(JSON.stringify({ data: "return from our handler", error: null }), {
      status: 200,
    });
  } catch (error) {
    console.error("Error processing request:", error);
    if (error instanceof Error)
      return new Response(JSON.stringify({ error: error }));
    return new Response(JSON.stringify({ error: "Unknown error" }));
  }
}
```

Now that we have our basic route handler, let's go back to our `SummaryForm.tsx` file and see if we can make a request to this endpoint.

But first let's navigate to `src/data/services` and create a new file called `summary-service.ts` and inside create a new service called `generateSummaryService.ts` and add the following code.

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

The following service allow us to call our newly created route handler located at `api/summarize` endpoint. It expects us to pass a `videoId` for the video that we would like to summarize.

Now, let's modify our `handleFormSubmit` to use our newly created service with the following code . Don't forget to import the `generateSummaryService` service.

```jsx
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

Now when you submit your form, inside our console log you should see the message being returned from our route handler.

![003-route-response](./images/003-route-response.gif)

Now that we know that out **Summary Form** and our **Route Handler** are connected we can move on to work on the logic responsible that will summarize our video.  

## Using Next.js Route Handler LangChain and Open AI To Create A Summary.

In this section we will take a look on how to create a video summary based on the video transcript.  

We will be using couple of service to help us accomplish this. 

Prerequesit:  you will need to have an account with **Open AI** if you don't have one, go [here](https://platform.openai.com/docs/introduction) and create one.


