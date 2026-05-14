# Personal-Portfolio / src/constants — How It Works

This folder holds **all data** for the page — projects list, experience timeline, social links — in one file: `constants/index.js`.

Pattern:

```js
export const myProjects = [
  {
    id: 1,
    title: "Project Title",
    description: "Short description.",
    subDescription: ["Bullet 1", "Bullet 2"],
    href: "https://...",
    logo: "/path.png",
    image: "/preview.jpg",
    tags: [
      { id: 1, name: "React", path: "/logos/react.png" },
    ],
  },
  // … more projects
];

export const experiences = [
  { title: "Engineer", company: "X", date: "2024 - Present", description: "..." },
  // ...
];

export const socials = [
  { name: "GitHub", href: "https://github.com/...", icon: GitHubIcon },
  // ...
];
```

### Why centralize data this way?

- **Separation of data & view**: components stay pure functions of props.
- **Easy to edit**: change a project title in one place, all UI updates.
- **Easy to migrate**: tomorrow you could replace this file with `fetch('/api/projects')` or a CMS query — components don't change.

### How it's consumed

```jsx
import { myProjects, experiences, socials } from '../constants';

myProjects.map(p => <Project key={p.id} {...p} />);
```

The spread `{...p}` passes every key of the project object as a prop. This works because the `<Project>` component destructures the same names. **It's clean but fragile**: if you rename a key in the data, every consumer breaks. For long-lived projects, prefer explicit `<Project title={p.title} ... />` (more typing, more safety).

### Patterns to extract

1. **`id` field for `key` prop**: every item gets a stable `id`. Never use array index as key in a real app.
2. **Nested arrays** (`tags`): perfectly fine, just `.map` again inside the renderer.
3. **`icon: GitHubIcon`**: storing a React component reference in data. Then `<entry.icon />` renders it. Pattern works for any component you want to swap.

That's it. This folder is intentionally tiny — it's just JS data.
