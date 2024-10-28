title: "What is Server-Side Rendering?"
date: 2024-10-28
author: GonzaloMB
tags: ["ssr", "backend", "frontend", "framework", "webdevelopment"]
---

![Untitled-2024-10-28-1858](https://github.com/user-attachments/assets/63886506-c37c-4118-93aa-8b19be306365)


#
- **`+page.server.js`**:

  ```javascript
  // Import any required modules, like databases or APIs
  export async function load({ params }) {
      const data = await fetchDataFromAPI(params.id);  // API or database call
      return { props: { data } }; // Send data as props to `+page.svelte`
  }

  ```
---

