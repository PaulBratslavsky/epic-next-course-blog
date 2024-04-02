# Epic Next JS 14 Tutorial: Learn Next JS by building a real-life project. Part 4: Building The Login and Registration Page

In the previous tutorial, we finished our **Home Page**, so we will build out our **Sign In** and **Sign Up** Pages and hook up the logic to allow us to sign in and sign up.

- Getting Started with the Project
- Building Out The Hero Section
- Building Out The Features Section, TopNavigation and Footer
- Building Out the Register and Sign In Page
- **Building Out the Dashboard page**
- Get Video Transcript with OpenAI Function
- Strapi CRUD permissions
- Search & pagination
- Backend deployment to Strapi Cloud and frontend deployment to Vercel

![001-dashboard-page](./images/001-dashboard-page.png)

Welcome to the next part of our React tutorial with Next.js. In the last post we finished our Signup & Signin Page with authentication using HTTPonly cookies, and saw how to protect our routes via Next.js middleware.

In this section we will be working on completing our **Dashboard** and **Profile Page** where we will take a look on how to upload files using Next.js server components.

Currently our **Dashboard** Page look like the following. Let's start by creating a `layout.tsx` page so we can give our page shared styling.

![002-current-state](./images/002-current-state.png)

Navigate to `src/app/dashboard` and create a file called `layout.tsx` and add the following code.

Your updated UI should look like the following.

![003-layout](./images/003-layout.png)

## Updating Top Header To Include Username and Logout Button

Currently our **Top Header** does not show the logged in user when we are logged in. Let's go ahead an update it.

Navigate to `src/app/components/custom/Header.tsx` make the following changes.

First, lets import our `getUserMeLoader` a function that we created in previous video to let us get our users data if they are logged in.

```jsx
import { getUserMeLoader } from "@/data/services/get-user-me-loader";
```

Next, let's call it insider our **Header** component with the following.

```jsx
const user = await getUserMeLoader();
console.log(user);
```

If you are loggedin you should see your user data in the console.

```js
{
  ok: true,
  data: {
    id: 3,
    username: 'testuser',
    email: 'testuser@email.com',
    provider: 'local',
    confirmed: true,
    blocked: false,
    createdAt: '2024-03-23T20:32:32.978Z',
    updatedAt: '2024-03-23T20:32:32.978Z'
  },
  error: null
}
```

We can use the `ok` key to conditionally render either our `Sign Up` button or the user's Name and Logout Button.

Before we can do that, let's import our **Logout** Button first, with the following.

```jsx
import { LogoutButton } from "./LogoutButton";
```

Now, let's create a simple component that will show the logout button and the users name. You can see the code in the following snippet.

```jsx
interface AuthUserProps {
  username: string;
  email: string;
}


export function LoggedInUser({ userData }: { readonly userData: AuthUserProps }) {
  return (
    <div className="flex gap-2">
      <Link
        href="/dashboard/account"
        className="font-semibold hover:text-primary"
      >
        {userData.username}
      </Link>
      <LogoutButton />
    </div>
  );
}
```

Now, let's update the following code.

```jsx
<div className="flex items-center gap-4">
  <Link href={ctaButton.url}>
    <Button>{ctaButton.text}</Button>
  </Link>
</div>
```

And replace with the new changes.

```jsx
<div className="flex items-center gap-4">
  {user.ok ? (
    <LoggedInUser userData={user.data} />
  ) : (
    <Link href={ctaButton.url}>
      <Button>{ctaButton.text}</Button>
    </Link>
  )}
</div>
```

The completed code in our `Header.tsx` file should look like the following.

```jsx
import Link from "next/link";

import { getUserMeLoader } from "@/data/services/get-user-me-loader";

import { Logo } from "@/components/custom/Logo";
import { Button } from "@/components/ui/button";
import { LogoutButton } from "./LogoutButton";

interface HeaderProps {
  data: {
    logoText: {
      id: number;
      text: string;
      url: string;
    };
    ctaButton: {
      id: number;
      text: string;
      url: string;
    };
  };
}

interface AuthUserProps {
  username: string;
  email: string;
}

export function LoggedInUser({
  userData,
}: {
  readonly userData: AuthUserProps;
}) {
  return (
    <div className="flex gap-2">
      <Link
        href="/dashboard/account"
        className="font-semibold hover:text-primary"
      >
        {userData.username}
      </Link>
      <LogoutButton />
    </div>
  );
}

export async function Header({ data }: Readonly<HeaderProps>) {
  const user = await getUserMeLoader();
  console.log(user);
  const { logoText, ctaButton } = data;
  return (
    <div className="flex items-center justify-between px-4 py-3 bg-white shadow-md dark:bg-gray-800">
      <Logo text={logoText.text} />
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

Nice, now when you are logged in, you should see the **username** & **Logout** Button.

Let's make another quick change in our `HeroSection.tsx` file found in `src/app/components/custom` folder.

The cool part about **React Server Components** is that they can be responsible for their own data. Let's updated so if the user is **Logged In** they will see the button to take them to the **Dashboard**.

Let's make the following changes.

```jsx
import Link from "next/link";
import { getUserMeLoader } from "@/data/services/get-user-me-loader";
import { StrapiImage } from "@/components/custom/StrapiImage";

interface ImageProps {
  id: number;
  url: string;
  alternativeText: string;
}

interface LinkProps {
  id: number;
  url: string;
  text: string;
}

interface HeroSectionProps {
  data: {
    id: number,
    __component: string,
    heading: string,
    subHeading: string,
    image: ImageProps,
    link: LinkProps,
  };
}

export async function HeroSection({ data }: Readonly<HeroSectionProps>) {
  const user = await getUserMeLoader();
  const { heading, subHeading, image, link } = data;

  const userLoggedIn = user.ok;
  const linkUrl = userLoggedIn ? "/dashboard" : link.url;

  return (
    <header className="relative h-[600px] overflow-hidden">
      <StrapiImage
        alt="Background"
        className="absolute inset-0 object-cover w-full h-full"
        height={1080}
        src={image.url}
        width={1920}
      />
      <div className="relative z-10 flex flex-col items-center justify-center h-full text-center text-white bg-black bg-opacity-20">
        <h1 className="text-4xl font-bold md:text-5xl lg:text-6xl">
          {heading}
        </h1>
        <p className="mt-4 text-lg md:text-xl lg:text-2xl">{subHeading}</p>
        <Link
          className="mt-8 inline-flex items-center justify-center px-6 py-3 text-base font-medium text-black bg-white rounded-md shadow hover:bg-gray-100"
          href={linkUrl}
        >
          {userLoggedIn ? "Dashboard" : link.text}
        </Link>
      </div>
    </header>
  );
}
```

Now our UI in the **Hero Section** should look like the following if the user is logged in.

![004-dashboard](./images/004-dashboard.png)

Now, let's work on our **Account** Page.

## Creating Our User Profile Page (Account Page)

Let's start by navigating to our `dashboard` folder and creating a folder called `account` with a `page.tsx` file.

We will add the following code.

```jsx
import { getUserMeLoader } from "@/data/services/get-user-me-loader";
// import { ProfileForm } from "@/components/forms/ProfileForm";
// import { ProfileImageForm } from "@/components/forms/ProfileImageForm";

export default async function AccountRoute() {
  const user = await getUserMeLoader();
  const userData = user.data;
  const userImage = userData?.image;

  return (
    <div className="grid grid-cols-1 lg:grid-cols-5 gap-4 p-4">
      Account Page
      {/* <ProfileForm data={userData} className="col-span-3" /> */}
      {/* <ProfileImageForm data={userImage} className="col-span-2" /> */}
    </div>
  );
}
```

I commented out the components that we are yet to create in order to get our app to render. Let's create our **ProfileForm** and **ProfileImageForm** components.

### Create a Form To Update Users Details

Lets' navigate to `src/app/components/forms` and create a file called `ProfileForm.tsx`.

Let's paste in the following code.

```jsx
"use client";
import React from "react";
import { cn } from "@/lib/utils";

import { SubmitButton } from "@/components/custom/SubmitButton"
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";

interface ProfileFormProps {
  id: string;
  username: string;
  email: string;
  firstName: string;
  lastName: string;
  bio: string;
  credits: number;
}

function CountBox({ text }: { readonly text: number }) {
  const style = "font-bold text-md mx-1";
  const color = text > 0 ? "text-primary" : "text-red-500";
  return (
    <div className="flex items-center justify-center h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-sm shadow-sm transition-colors file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none">
      You have<span className={cn(style, color)}>{text}</span>credit(s)
    </div>
  );
}

export function ProfileForm({
  data,
  className,
}: {
  readonly data: ProfileFormProps;
  readonly className?: string;
}) {

  return (
    <form
      className={cn("space-y-4", className)}>
      <div className="space-y-4 grid ">
        <div className="grid grid-cols-3 gap-4">
          <Input
            id="username"
            name="username"
            placeholder="Username"
            defaultValue={data.username || ""}
            disabled
          />
          <Input
            id="email"
            name="email"
            placeholder="Email"
            defaultValue={data.email || ""}
            disabled
          />
          <CountBox text={data.credits} />
        </div>

        <div className="grid grid-cols-2 gap-4">
          <Input
            id="firstName"
            name="firstName"
            placeholder="First Name"
            defaultValue={data.firstName || ""}
          />
          <Input
            id="lastName"
            name="lastName"
            placeholder="Last Name"
            defaultValue={data.lastName || ""}
          />
        </div>
        <Textarea
          id="bio"
          name="bio"
          placeholder="Write your bio here..."
          className="resize-none border rounded-md w-full h-[224px] p-2"
          defaultValue={data.bio || ""}
          required
        />
      </div>
      <div className="flex justify-end">
        <SubmitButton text="Update Profile" loadingText="Saving Profile" />
      </div>
    </form>
  );
}

```

Since we are using a new **Shadcn UI** component `Textarea` let's install it with the following.

```bash
npx shadcn-ui@latest add textarea
```

Now, let's uncomment our **ProfileForm** in our `dashboard/account/page.tsx` file.

```jsx
import { getUserMeLoader } from "@/data/services/get-user-me-loader";
import { ProfileForm } from "@/components/forms/ProfileForm";
// import { ProfileImageForm } from "@/components/forms/ProfileImageForm";

export default async function AccountRoute() {
  const user = await getUserMeLoader();
  const userData = user.data;
  const userImage = userData?.image;

  return (
    <div className="grid grid-cols-1 lg:grid-cols-5 gap-4 p-4">
      Account Page
      <ProfileForm data={userData} className="col-span-3" />
      {/* <ProfileImageForm data={userImage} className="col-span-2" /> */}
    </div>
  );
}
```

Restart the app and you Next.js frontend and you should see the following.

![005-account-form](./images/005-account-form.png)

You should notice two things, one we are not getting our users **First Name**, **Last Name**, **Bio** or number of credits they have left.

And second, we are not able to submit the form because we did not implement the form logic via server action yet. We will do that next. But first let's update our user schema in Strapi.

### Updating User Data Schema In Our Backend

Open up your Strapi backend and navigate to the `content-builder` and choose the user collection type.

![006-user-admin](./images/006-user-admin.png)

Let's add the following fields.

| Name      | Field  | Type       | Advanced Settings         |
| --------- | ------ | ---------- | ------------------------- |
| firstName | Text   | Short Text |                           |
| lastName  | Text   | Short Text |                           |
| bio       | Text   | Long Text  |                           |
| credits   | Number | Integer    | Set default value to be 0 |

We will add the credits manually for new users when they sign in, but their default starting credits should be `0`.

Once you are done you should have the following new fields.

![007-user-fields](./images/007-user-fields.png)

Now, let's manually update our user's information so we can check if we are getting it in our frontend.

![008-user-updated](./images/008-user-updated.png)

When you navigate to your `Account` page in your frontend you should see the following output.

![updated-user-ui](./images/009-updated-user-ui.png)

Now, let's move on to the form update using **server action** and learn about **revalidatePath**.

### Updating User Date With Server Actions

First let's create our `updateProfileAction` that will be responsible for handling our form submission.

Navigate to `src/data/actions` and create a new file called `profile-actions` and paste in the following code.

```jsx
"use server";
import qs from "qs";

export async function updateProfileAction(
  userId: string,
  prevState: any,
  formData: FormData
) {
  const rawFormData = Object.fromEntries(formData);

  const query = qs.stringify({
    populate: "*",
  });

  const payload = {
    firstName: rawFormData.firstName,
    lastName: rawFormData.lastName,
    bio: rawFormData.bio,
  };

  console.log("updateProfileAction", userId);
  console.log("############################");
  console.log(payload);
  console.log("############################");

  return {
    ...prevState,
    message: "Profile Updated",
    data: payload,
    strapiErrors: null,
  };
}
```

We have created actions before, so not much new here except with one small addition, notice that we can access to `userId` which we are getting as one of the arguments.

Let's implement this action in our `ProfileForm` and see how we are passing the `userId` to our action.

Navigate back to your `ProfileForm.tsx` file and make the following changes.

First, let's import our action with the following.

```jsx
import { useFormState } from "react-dom";
import { updateProfileAction } from "@/data/actions/profile-actions";
```

Next, let's create initial state for our `useFormState`.

```jsx
const INITIAL_STATE = {
  data: null,
  strapiErrors: null,
  message: null,
};
```

I am not going to focus on form validation with **Zod** since we already covered this in previous sections. And can be a great extra challenge for you to explore on your own.

But we are going to import our **StrapiErrors** component and handle those.

```jsx
import { StrapiErrors } from "@/components/custom/StrapiErrors";
```

Before using the `useFormState` like we did in previous sections, let's tale a look how we can bind additional data that we would like to pass to our server actions.

Add the following line of code.

```jsx
const updateProfileWithId = updateProfileAction.bind(null, data.id);
```

We can use the `bind` method to add new data, that we will have access to inside our server action.

This is how we are setting our `userId` in order to be able to access it from our `updateProfileAction` server action.

You can read more about it in the Next.js documentation [here](https://nextjs.org/docs/app/building-your-application/data-fetching/server-actions-and-mutations#passing-additional-arguments).

Now finally let's use our `useFormState` hook so we can have access to our returned data from our server actions.

```jsx
const [formState, formAction] = useFormState(
  updateProfileWithId,
  INITIAL_STATE
);
```

Let's update our `form` tag with the following.

```jsx
  <form
    className={cn("space-y-4", className)}
    action={formAction}
  >
```

And don't forget to add our **StrapiErrors** component.

```jsx
<div className="flex justify-end">
    <SubmitButton text="Update Profile" loadingText="Saving Profile" />
</div>
<StrapiErrors error={formState?.strapiErrors} />
```

The final code should look like the following inside your `ProfileForm.tsx` file.

```jsx
"use client";
import React from "react";
import { cn } from "@/lib/utils";

import { useFormState } from "react-dom";
import { updateProfileAction } from "@/data/actions/profile-actions";

import { SubmitButton } from "@/components/custom/SubmitButton";
import { Input } from "@/components/ui/input";
import { Textarea } from "@/components/ui/textarea";
import { StrapiErrors } from "@/components/custom/StrapiErrors";

const INITIAL_STATE = {
  data: null,
  strapiErrors: null,
  message: null,
};

interface ProfileFormProps {
  id: string;
  username: string;
  email: string;
  firstName: string;
  lastName: string;
  bio: string;
  credits: number;
}

function CountBox({ text }: { readonly text: number }) {
  const style = "font-bold text-md mx-1";
  const color = text > 0 ? "text-primary" : "text-red-500";
  return (
    <div className="flex items-center justify-center h-9 w-full rounded-md border border-input bg-transparent px-3 py-1 text-sm shadow-sm transition-colors file:border-0 file:bg-transparent file:text-sm file:font-medium placeholder:text-muted-foreground focus-visible:outline-none">
      You have<span className={cn(style, color)}>{text}</span>credit(s)
    </div>
  );
}

export function ProfileForm({
  data,
  className,
}: {
  readonly data: ProfileFormProps;
  readonly className?: string;
}) {
  const updateProfileWithId = updateProfileAction.bind(null, data.id);

  const [formState, formAction] = useFormState(
    updateProfileWithId,
    INITIAL_STATE
  );

  return (
    <form
      className={cn("space-y-4", className)}
      action={formAction}
    >
      <div className="space-y-4 grid ">
        <div className="grid grid-cols-3 gap-4">
          <Input
            id="username"
            name="username"
            placeholder="Username"
            defaultValue={data.username || ""}
            disabled
          />
          <Input
            id="email"
            name="email"
            placeholder="Email"
            defaultValue={data.email || ""}
            disabled
          />
          <CountBox text={data.credits} />
        </div>

        <div className="grid grid-cols-2 gap-4">
          <Input
            id="firstName"
            name="firstName"
            placeholder="First Name"
            defaultValue={data.firstName || ""}
          />
          <Input
            id="lastName"
            name="lastName"
            placeholder="Last Name"
            defaultValue={data.lastName || ""}
          />
        </div>
        <Textarea
          id="bio"
          name="bio"
          placeholder="Write your bio here..."
          className="resize-none border rounded-md w-full h-[224px] p-2"
          defaultValue={data.bio || ""}
          required
        />
      </div>
      <div className="flex justify-end">
        <SubmitButton text="Update Profile" loadingText="Saving Profile" />
      </div>
      <StrapiErrors error={formState?.strapiErrors} />
    </form>
  );
}

```

Let's test it and see if we are able to console log our changes before making the API call to Strapi.

If you check your terminal you will see the following console message.

```js
updateProfileAction 3
############################
{
  firstName: 'Paul',
  lastName: 'Brats',
  bio: 'I made this update'
}
############################
```

Notice we are getting our **userId** and our data that we want to update.

Now let's go ahead an implement the logic that will send this data to **Strapi**.

But first let's navigate to `src/data/services` and create a new service called `mutate-data.ts` and import the following code.

```ts
import { getAuthToken } from "./get-token";
import { getStrapiURL } from "@/lib/utils";

export async function mutateData(method: string, path: string, payload?: any) {
  const baseUrl = getStrapiURL();
  const authToken = await getAuthToken();
  const url = new URL(path, baseUrl);

  if (!authToken) throw new Error("No auth token found");

  try {
    const response = await fetch(url, {
      method: method,
      headers: {
        "Content-Type": "application/json",
        Authorization: `Bearer ${authToken}`,
      },
      body: JSON.stringify({ ...payload }),
    });
    const data = await response.json();
    return data;
  } catch (error) {
    console.log("error", error);
    throw error;
  }
}
```

Here we are just using fetch to submit our data, and to make it more flexible and reusable we are passing `path` and `payload` as arguments.

Let's use in on our `profile-actions.ts` file.

Let's make the following change.

```jsx
"use server";
import qs from "qs";
import { mutateData } from "@/data/services/mutate-data";
import { flattenAttributes } from "@/lib/utils";

export async function updateProfileAction(
  userId: string,
  prevState: any,
  formData: FormData
) {
  const rawFormData = Object.fromEntries(formData);

  const query = qs.stringify({
    populate: "*",
  });

  const payload = {
    firstName: rawFormData.firstName,
    lastName: rawFormData.lastName,
    bio: rawFormData.bio,
  };

  const responseData = await mutateData(
    "PUT",
    `/api/users/${userId}?${query}`,
    payload
  );

  if (!responseData) {
    return {
      ...prevState,
      strapiErrors: null,
      message: "Ops! Something went wrong. Please try again.",
    };
  }

  if (responseData.error) {
    return {
      ...prevState,
      strapiErrors: responseData.error,
      message: "Failed to Register.",
    };
  }

  const flattenedData = flattenAttributes(responseData);

  return {
    ...prevState,
    message: "Profile Updated",
    data: flattenedData,
    strapiErrors: null,
  };
}
```

Now, try to update your form and you will get the `forbidden` message.

![010-forbidden](./images/010-forbidden.png)

In order for this to work, we need to give permission in Strapi's Admin to allow us to make the changes.

![add-user-permission](./images/011-add-user-permission.png)

**note:** One thing to keep in mind, you should take an additional step to protect your **User** route by creating an additional `policy` that will only allow you to update your own user date.

We will cover this as an addendum after we complete this serries.

But let's try to update our profile and see if it works.

![012-form-update](./images/012-form-update.gif)

Now that we can update our profile. Let's take a look how we can upload files in Next.js.

### Uploading Files In Next.js Using Server Actions.

We are now going to focus on how to handle file upload in **Next.js** with **Server Actions**. But before we can do that, let's create an **ImagePicker** component.

Navigate to `src/components/custom` and create a file called `ImagePicker.tsx` and paste the following code.

```jsx
"use client";
import React, { useState, useRef } from "react";
import { StrapiImage } from "./StrapiImage";

import { Input } from "@/components/ui/input";
import { Label } from "@/components/ui/label";

interface ImagePickerProps {
  id: string;
  name: string;
  label: string;
  showCard?: boolean;
  defaultValue?: string;
}

function generateDataUrl(file: File, callback: (imageUrl: string) => void) {
  const reader = new FileReader();
  reader.onload = () => callback(reader.result as string);
  reader.readAsDataURL(file);
}

function ImagePreview({ dataUrl }: { readonly dataUrl: string }) {
  return (
    <StrapiImage
      src={dataUrl}
      alt="preview"
      height={200}
      width={200}
      className="rounded-lg w-full object-cover"
    />
  );
}

function ImageCard({
  dataUrl,
  fileInput,
}: {
  readonly dataUrl: string;
  readonly fileInput: React.RefObject<HTMLInputElement>;
}) {
  const imagePreview = dataUrl ? <ImagePreview dataUrl={dataUrl} /> : <p>No image selected</p>;

  return (
    <div className="w-full relative">
      <div className=" flex items-center space-x-4 rounded-md border p-4">
        {imagePreview}
      </div>
      <button
        onClick={() => fileInput.current?.click()}
        className="w-full absolute inset-0"
        type="button"
      ></button>
    </div>
  );
}

export default function ImagePicker({
  id,
  name,
  label,
  defaultValue,
}: Readonly<ImagePickerProps>) {
  const fileInput = useRef<HTMLInputElement>(null);
  const [dataUrl, setDataUrl] = useState<string | null>(defaultValue ?? null);

  const handleFileChange = (e: React.ChangeEvent<HTMLInputElement>) => {
    const file = e.target.files?.[0];
    if (file) generateDataUrl(file, setDataUrl);
  };

  return (
    <React.Fragment>
      <div className="hidden">
        <Label htmlFor={name}>{label}</Label>
        <Input
          type="file"
          id={id}
          name={name}
          onChange={handleFileChange}
          ref={fileInput}
          accept="image/*"
        />
      </div>
      <ImageCard dataUrl={dataUrl ?? ""} fileInput={fileInput} />
    </React.Fragment>
  );
}

```

This components is responsible for letting the user select an image in the form.

Now, lets create our **ProfileImageForm** that will utilize this component.

Navigate to `src/components/forms` and create a file called `ProfileImageForm.tsx` and paste in the following code.

```jsx
"use client";
import React from "react";
import { useFormState } from "react-dom";
import { cn } from "@/lib/utils";

import { uploadProfileImageAction } from "@/data/actions/profile-actions";

import { SubmitButton } from "@/components/custom/SubmitButton";
import ImagePicker from "@/components/custom/ImagePicker";
import { ZodErrors } from "@/components/custom/ZodErrors";
import { StrapiErrors } from "@/components/custom/StrapiErrors";

interface ProfileImageFormProps {
  id: string;
  url: string;
  alternativeText: string;
}

const initialState = {
  message: null,
  data: null,
  strapiErrors: null,
  zodErrors: null,
};

export function ProfileImageForm({
  data,
  className,
}: {
  data: Readonly<ProfileImageFormProps>;
  className?: string;
}) {

  const uploadProfileImageWithIdAction = uploadProfileImageAction.bind(null, data?.id);
  
  const [formState, formAction] = useFormState(
    uploadProfileImageWithIdAction,
    initialState
  );

  return (
    <form
      className={cn("space-y-4", className)}
      action={formAction}
    >
      <div className="">
        <ImagePicker
          id="image"
          name="image"
          label="Profile Image"
          defaultValue={data?.url || ""}
        />
        <ZodErrors error={formState.zodErrors?.image} />
        <StrapiErrors error={formState.strapiErrors} />
      </div>
      <div className="flex justify-end">
        <SubmitButton text="Update Image" loadingText="Saving Image" />
      </div>
    </form>
  );
}

```

Now let's navigate to `profile-actions.ts` and update the file with the following code.

```jsx
"use server";
import { z } from "zod";
import qs from "qs";

import { getUserMeLoader } from "@/data/services/get-user-me-loader";
import { mutateData } from "@/data/services/mutate-data";
import { flattenAttributes } from "@/lib/utils";

import { fileDeleteService, fileUploadService } from "@/data/services/file-service";

import { revalidatePath } from "next/cache";

export async function updateProfileAction(
  userId: string,
  prevState: any,
  formData: FormData
) {
  const rawFormData = Object.fromEntries(formData);

  const query = qs.stringify({
    populate: "*",
  });

  const payload = {
    firstName: rawFormData.firstName,
    lastName: rawFormData.lastName,
    bio: rawFormData.bio,
  };

  const responseData = await mutateData(
    "PUT",
    `/api/users/${userId}?${query}`,
    payload
  );

  if (!responseData) {
    return {
      ...prevState,
      strapiErrors: null,
      message: "Ops! Something went wrong. Please try again.",
    };
  }

  if (responseData.error) {
    return {
      ...prevState,
      strapiErrors: responseData.error,
      message: "Failed to Register.",
    };
  }

  const flattenedData = flattenAttributes(responseData);

  return {
    ...prevState,
    message: "Profile Updated",
    data: flattenedData,
    strapiErrors: null,
  };
}


const MAX_FILE_SIZE = 5000000;

const ACCEPTED_IMAGE_TYPES = [
  "image/jpeg",
  "image/jpg",
  "image/png",
  "image/webp",
];

const imageSchema = z.object({
  image: z
    .any()
    .refine((file) => {
      if (file.size === 0 || file.name === undefined) return false;
      else return true;
    }, "Please update or add new image.")

    .refine(
      (file) => ACCEPTED_IMAGE_TYPES.includes(file?.type),
      ".jpg, .jpeg, .png and .webp files are accepted."
    )
    .refine((file) => file.size <= MAX_FILE_SIZE, `Max file size is 5MB.`),
});

export async function uploadProfileImageAction(
  imageId: string,
  prevState: any,
  formData: FormData
) {
  const user = await getUserMeLoader();
  const userId = user.data.id;

  if (!userId) throw new Error("User not found");

  const data = Object.fromEntries(formData);

  const validatedFields = imageSchema.safeParse({
    image: data.image,
  });


  if (!validatedFields.success) {
    return {
      ...prevState,
      zodErrors: validatedFields.error.flatten().fieldErrors,
      strapiErrors: null,
      data: null,
      message: "Invalid Image",
    };
  }

  if (imageId) await fileDeleteService(imageId);

  const fileUploadResponse = await fileUploadService(data.image);

  if (!fileUploadResponse) {
    return {
      ...prevState,
      strapiErrors: null,
      zodErrors: null,
      message: "Ops! Something went wrong. Please try again.",
    };
  }

  if (fileUploadResponse.error) {
    return {
      ...prevState,
      strapiErrors: fileUploadResponse.error,
      zodErrors: null,
      message: "Failed to Upload File.",
    };
  }
  const updatedImageId = fileUploadResponse[0].id;
  const payload = { image: updatedImageId };

  const updateImageResponse = await mutateData( "PUT",`/api/users/${userId}`,payload);
  const flattenedData = flattenAttributes(updateImageResponse);
  revalidatePath("/dashboard/account");
  return { ...prevState, data: flattenedData, zodErrors: null, strapiErrors: null, message: "Image Uploaded"};
}

```

The above server action handles the following steps

1. First checks if the user is logged in and identifies their account.
2. It then checks the image the user selected to make sure it's a valid image type (like JPEG or PNG) and not too large in file size.
3. If the image is valid, and there was previously an image, the old image is removed.
4. The new image is then uploaded to the server and associated with the user's profile.
5. The application updates the user's profile with this new image. If there's an error, it informs the user.

In essence, this code is about two main actions on a user's profile in an application: updating personal details and changing the profile picture. 

It makes sure that the data is valid via **Zod** and communicates with the server to store these changes, providing feedback to the user based on whether these actions were successful or not.

To check if the image is valid we are using `refine` method in **Zod** that allows us to create custom validation logic.

Let's briefly look over the use of `refine` in the code below.

``` ts
const imageSchema = z.object({
  image: z
    .any()
    .refine((file) => {
      if (file.size === 0 || file.name === undefined) return false;
      else return true;
    }, "Please update or add new image.")

    .refine(
      (file) => ACCEPTED_IMAGE_TYPES.includes(file?.type),
      ".jpg, .jpeg, .png and .webp files are accepted."
    )
    .refine((file) => file.size <= MAX_FILE_SIZE, `Max file size is 5MB.`),
});
```

**Why Use refine**
Flexibility and Precision: refine allows for custom validations beyond basic type checks. This means you can implement complex, tailored criteria for data validity.

User Feedback: By specifying an error message with each refine, you provide clear, actionable feedback to the user, improving the user experience by guiding them on how to correct their input.

Composition: Multiple refine validations can be chained together, allowing for a sequence of checks that are both comprehensive and readable.

You can learn more about using `refine` [here](https://zod.dev/?id=refine). 


Finally let's create our two services,  **fileDeleteService** & **fileUploadService**:  they will be responsible for handling deleting an image that already exist and uploading a new one.

Navigate to `src/data/services` and create a file named `file-service.ts` and paste in the following code.

To learn more about `file upload` in Strapi take a look here in the [docs](https://docs.strapi.io/dev-docs/plugins/upload#endpoints).

```ts
import { getAuthToken } from "@/data/services/get-token";
import { mutateData } from "@/data/services/mutate-data";
import { flattenAttributes } from "@/lib/utils";
import { getStrapiURL } from "@/lib/utils";

export async function fileDeleteService(imageId: string) {
  const authToken = await getAuthToken();
  if (!authToken) throw new Error("No auth token found");

  const data = await mutateData("DELETE", `/api/upload/files/${imageId}`);
  const flattenedData = flattenAttributes(data);

  return flattenedData;
}

export async function fileUploadService(image: any) {
  const authToken = await getAuthToken();
  if (!authToken) throw new Error("No auth token found");

  const baseUrl = getStrapiURL();
  const url = new URL("/api/upload", baseUrl);

  const formData = new FormData();
  formData.append("files", image, image.name);

  try {
    const response = await fetch(url, {
      headers: { Authorization: `Bearer ${authToken}` },
      method: "POST",
      body: formData,
    });

    const dataResponse = await response.json();

    return dataResponse;
  } catch (error) {
    console.error("Error uploading image:", error);
    throw error;
  }
}
```

Now that our frontend is ready, let's add the `image` field in our user collection type in Strapi Admin.

![013-add-image](./images/013-add-image.png)

Navigate to **Content Type Builder** click on the **User** collection type and click on the **add another field to this collection** button.

![014-select-media](./images/014-select-media.png)

Select the `media` field.

Make sure to name it `image` and select `Single media` option and then navigate to `Advanced Settings` tab.

In the advance setting tabs, configure  `allowed file types` only to include images, once done click on the `Finish` button.

Now, add a image to your user. 

![017-upload-image](./images/017-upload-image.png)

Refresh you frontend application and you should now see your newly added user image.

![018-user-image](./images/018-user-image.png) 

Before we can test if our image uploader works, let's change our setting in Strapi to enable file upload and delete.

You can find these options under `Settings` => `USERS & PERMISSIONS PLUGIN` => `Roles` => `Authenticated` => `Upload`.

Check both `upload` and `destroy` boxes.

![019-upload](./images/019-upload.png)

Let's test out our upload functionality.

![20-test-upload](./images/20-test-upload.gif)

Nice, we now have our file upload working. 

In the next post, we will take a look how to handle our video summary generation.

## Conclusion

Nice, we completed our initial **Dashboard** layout with a **Account** section where the user is able to update their `first name`, `last name`, `bio` and `image`.

We covered how to handle file upload using server actions, by this point you should be starting to feel more comfortable on how to work with `forms` and `server actions` in Next.js.

In the next post, we will start working on our main feature which will allow us to summarize our youtube videos.

See you in the next one.

P.S.  If you made it this far thank you.  I really appreciate your support.  I did my best do dillgence, but if you find errors, shared them in the comments bellow.  