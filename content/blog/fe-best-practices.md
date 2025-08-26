---
date: '2024-12-28T10:30:00-04:00'
draft: false
title: 'FE best practices'
description: "My two cents on how to build scalable FE applications"
---

**DISCLAIMER:** As predicted, I disagree with a lot of what I wrote here earlier 😅. With that being said, I still want to keep it up since there is still some useful information, and it's always fun to see how far you've come. 

Scalable front-end code is an expansive topic; of course, there are topics I can't cover here, such as effective testing, development philosophies, monorepos, and much more. This article focuses on practical strategies for scaling front-end code in terms of architecture, maintainability, and technologies. My primary goal is to leave you with an idea of WHAT you should be researching and looking into, not necessarily explaining the nitty-gritty of all topics. This guide is meant to be framework agnostic, and my advice here should be transferable to any framework.

It doesn't need to be said, but I'll say it anyway - this is my perspective based on what I know right now. I’m always learning and evolving, so I’m sure I’ll look back on this article in a few months and laugh.

# Choosing the Right Technology

Technologies are always evolving, so I won't tell you what the best technology is right now. In web development, something considered great today can be outdated and replaced by tomorrow. It's important to know what types of technology to focus your attention on and how to evaluate the technology.


## Component Libraries

It's too much overhead to build out your own component library; for most of us, our time is better spent elsewhere. The challenge with building your own component library lies in ensuring your content is accessible. This is no easy task, and accessibility should not be something you mess with; everyone deserves to surf the web with minimal annoyances. When choosing your component library, you must evaluate your project's needs and how much low-level control you need over styling. If you're making a marketing website, you probably need a lot of control over styles, but if you're making a web app, MUI is probably enough.

As a side note, try to pick out component libraries that don't use CSS-in-JS - it's terrible for performance, so maybe MUI isn't the best choice... I'd also recommend finding a tool that integrates well with Tailwind CSS (like Shadcn).

## Frameworks

Frameworks should solve the problem that you are facing. Let's take [Remix](https://remix.run/) and Shopify. Shopify likely uses Remix on the front-end because of its fine-grained control over data fetching and rendering, whether on the client or server side. Shopify applications are fetching mountains of data, and it has to be optimized to render on the client side for interactivity and on the server side to optimize load times and SEO. Additionally, Remix is loosely coupled to the JSX you write; in fact, I'd say almost not at all, and this allows Shopify to easily migrate and switch out Remix for other technologies when the time comes, which is part of their philosophy. The architects at Shopify chose Remix not because it's perfect, but because it solves their specific problem.

## State Management

Before even thinking about the state, break down your app as though it were a CLI. Don't get distracted by buttons or input fields. Focus on the ACTIONS a user can take on your app. Flow chart and diagram how a user traverses and uses your app. State management has existed for decades before any front-end frameworks were created. If you understand the actions a user takes and the consequences of those actions, then the state management solution will be clear. For example, if your state is very reactive, needs validation, or influences cause API calls, maybe you want a harder-hitting state machine rather than Redux?

The technology you use should follow the principle: "low coupling, high cohesion." Think about software as just boxes inside of boxes - these boxes should exist on their own and should never rely on another box to function. However, when these boxes work together, they can form software to achieve a higher goal. If the technology you choose forces you to be coupled to it (cough, cough Next.js), maybe it isn't the right fit...


{{< figure src="low-coupling-high-cohension.png" width="100%" alt="low coupling high cohesion" >}}
<sub><sup>Image credit: [Feature-Sliced Design](https://feature-sliced.design/docs/reference/slices-segments)</sup></sub>


# Folder Structure

This is where your application basically becomes Angular or Ember.js, no matter what framework you use, both of which propose a similar organizational methodology with differing languages.

## Feature-Sliced Design

[Feature-sliced](https://feature-sliced.design/) design is a methodology for structuring front-end projects by breaking them into three well-defined layers, focusing on scalability and maintainability. The first layer's key components are the following: The app, which includes application-level logic like routing, entry points, and global configurations. Features, which are self-contained modules representing distinct user actions or app capabilities (e.g., login, search). Entities, which are business entities or domain objects (e.g., User, Product) with associated logic and data. Shared, which includes reusable utilities, libraries, or constants that can be used across the entire app. UI/Components, which are basic, reusable presentational components without application-specific logic. The methodology emphasizes grouping code by feature rather than type, making it clear which code belongs where and ensuring that shared resources are truly shared across the app. The second layer is slices, which are key parts of a feature. The final layer is segments. When I create my segments, I take inspiration from [Bulletproof React](https://github.com/alan2207/bulletproof-react). Bullet-proof React suggests breaking down feature slices into utils, API, UI, config, etc.

Remember when I mentioned that good software thrives on low coupling and high cohesion? Feature-sliced design embodies this principle. Each feature is self-contained, functioning independently while relying only on the "shared" directory. At the same time, these features come together cohesively to form your application. If you've ever navigated a messy, unorganized codebase, you understand the frustration of tracking down a component. By organizing code by features, you make everything more intuitive and accessible.

# Design Patterns
I'd recommend reading engineering blogs from companies, such as Google, Datadog, and Patterns.Dev, and my favourite, Figma, to learn design patterns and study how battle-tested open source tools propose structuring code. I'll focus on three design patterns that I've implemented, which have tremendously improved my code quality.

## Presenter, Container
Separating components into presenters and containers enhances maintainability, reusability, testability, and scalability. Presenter components focus on how the UI is rendered, handling only visual aspects, while container components manage what is rendered by handling logic, state, and data fetching. This separation ensures each component has a single responsibility, making the code easier to understand and modify. Presenter components become reusable across contexts since they are not tied to specific logic, while container components focus solely on data handling. Testing is streamlined, as visual and logical concerns can be addressed independently. This structure aligns with the single-responsibility principle, enabling smoother scaling as the application grows.

{{< figure src="/container-presenter.png" width="100%" alt="Container Presenter Pattern" >}}
<sub><sup>Image credit: [Luke from Medium.com](https://medium.com/generic-ui/the-new-approach-to-the-container-presenter-pattern-in-angular-dac60ca1b65e)</sup></sub>

## File Naming Conventions
Personally, I follow [Angular's naming conventions](https://angular.dev/style-guide), which are phenomenal and have boosted my dev-ex substantially. The Angular documentation proposes adding a suffix to your file name that describes its purpose. For example, if it's a component, add a `.component`, if it's a utility, add a `.util`, if it's a model, add a `.model` and so on! This simple convention reduces decision fatigue and ensures consistency across your project. Furthermore, finding files in your IDE is much easier - search using the suffix. Additionally, code reviews take much less time since the reviewer can interpret the purpose of a file without opening it. By using descriptive suffixes like `.component`, `.service`, or `.util`, the developer is prompted to think about the file's purpose and avoid mixing unrelated logic.

## Leverage Custom Hooks

Custom hooks make your code more modular, reusable, and readable—when implemented correctly (refer to the React hook rules for best practices). Hooks allow you to abstract away how something is done, enabling you to focus on what is being accomplished, similar to a reducer. They're especially valuable for managing features that trigger multiple side effects. Take a filter feature, for instance: each time the user interacts with it, your program might either make an API call or sort data client-side, depending on the architecture. Instead of embedding this logic in your component, you can abstract it into a custom hook, providing a clean and reusable interface for your components. However, avoid making hooks overly specific, as this increases coupling and complicates future migrations or refactors.


# Design Systems
Learn. Figma.

In my opinion, all front-end devs should also be knowledgeable about UI/UX and be able to build out or extend a design system. UI/UX engineers offer a very valuable perspective on UX best practices, but a front-end dev can offer a technical point of view. However, if you, as a front-end dev, know the basics of UI/UX, then you can effectively work with designers to create an optimal solution. I'd recommend looking at [cuHacking](https://www.figma.com/design/wc1JOWR48tBNkjcjwY3AzB/%E2%8C%A8%EF%B8%8F-cuHacking-Design-System?node-id=605-271&t=PWNXDGvjiyuKR3X9-0) for how we implemented our design system. While not all design systems need to follow this exact approach, we adapted the Radix design system philosophy to better meet the unique needs of our platform. If you want to learn the basics of Design systems, Figma has an article on exactly that, which I would recommend reading to get a foundational idea of how to build a design system.

A standardized design system reduces decision fatigue, and with the effective use of Figma and Token Studio for Figma, you can create a 1-1 relationship with Figma and code.

Design systems are similar to SQL databases. SQL databases are notoriously rigid, but they allow devs to operate at high velocity. Similarly, following a design system leads developers and designers to move with incredible velocity. Just like an SQL database, as your design system evolves, you can extend and refine it to fit your needs better.

# Final Remarks

As the web development system constantly evolves so will our perceptions on what makes front-end code truly scalable. At the end of the day, this article is just theory and nothing more. Everyone reading this should always try and experiment with different ideologies, frameworks, and methods to come up with their own set of best practices. 
