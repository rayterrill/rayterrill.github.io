---
title:  "NextJS Next-Themes Dark/Light Mode with App Router"
tags: [nextjs]
---

For some nextjs sites, it's nice to have dark/light mode support. next-themes makes this relatively easy, although it's not super intuitive how to set this up.

1. Build a new nextjs project, install next-themes, and start the dev server:
     ```
     npx create-next-app@latest
     cd <yourprojectname>
     npm i next-themes
     npm run dev
     ```
2. (Optional) With a new project, I usually remove the styles created by create-next-app in the globals.css to reset things back to the default tailwind styles:
    ```
    @tailwind base;
    @tailwind components;
    @tailwind utilities;
    ```
3. Under your app folder, create a new providers.tsx file that will contain your providers, and add the new ThemeProvider from next-themes. This is useful so you can make this is client component with access to client context without requiring that your layout become a client component. Note that the ThemeProvider will set the theme using a class attribute (`attribute="class"`) and has system theme sync enabled.
    ```
    'use client'
    
    import { ThemeProvider } from "next-themes"
    
    export const Providers = ({children} : {children: React.ReactNode}) => {
        return (
            <ThemeProvider attribute="class" enableSystem>
                {children}
            </ThemeProvider>
        )
    }
    ```
4. In your tailwind.config.ts file, add a new property `darkMode: "class",` under the Config block. This tells tailwind to enable darkmode based on a class attribute (which will be provided by next-themes):
    ```
    import type { Config } from "tailwindcss";
    
    const config: Config = {
      content: [
        "./src/pages/**/*.{js,ts,jsx,tsx,mdx}",
        "./src/components/**/*.{js,ts,jsx,tsx,mdx}",
        "./src/app/**/*.{js,ts,jsx,tsx,mdx}",
      ],
      theme: {
        container: {
            center: true,
            padding: {
                DEFAULT: "1rem",
                md: "1.5rem",
                lg: "2rem",
            },
        },
      },
      darkMode: "class",
      plugins: [],
    };
    export default config;
    ```
5. In your layout, consume the new providers.tsx component you created in step 2.
    ```
    import type { Metadata } from "next";
    import { Inter } from "next/font/google";
    import "./globals.css";
    import { Providers } from './providers'
    import Link from 'next/link'
    
    const inter = Inter({ subsets: ["latin"] });
    
    export const metadata: Metadata = {
      title: "Create Next App",
      description: "Generated by create next app",
    };
    
    export default function RootLayout({
      children,
    }: Readonly<{
      children: React.ReactNode;
    }>) {
      return (
        <html lang="en" suppressHydrationWarning>
            <body className={inter.className}>
              <Providers>
                {children}
              </Providers>
            </body>
        </html>
      );
    }
    ```

At this point, you should have dark/light mode with system sync working!