# Workspace Context: Obsidian Vault & Jekyll Blog

This workspace (`mhdomer.github.io`) is a dual-purpose repository:
1. **Local Obsidian Vault**: The user uses Obsidian as their local knowledge base.
2. **Jekyll Blog**: The site is built with `jekyll-theme-chirpy` and deployed to GitHub Pages.

## CRITICAL RULES FOR AI ASSISTANTS:
1. **Respect `.gitignore`**: Several folders at the root (e.g., `Semester 6`, `FYP`, `INTERN LI`, `MAIN`, `SCS-C03 Cloud Security Speciality notes`, `RoadMaps and Career`, `Projects Idea and implementation`) and the `.obsidian` configuration folder are strictly ignored by Git. 
2. **Never modify or delete raw personal notes**: Only interact with published blog content located inside the `_posts/` folder unless explicitly requested by the user.
3. **Commit structure**: The `_posts/` folder contains subdirectories that mirror the Jekyll frontmatter categories (e.g., `OSCP Certification`, `DevSecOps`). Maintain this structure when adding or editing posts.
4. **Git Tracking**: When making commits, ensure ignored folders (like `.obsidian`) are NEVER staged or tracked. Only commit valid blog updates and structural changes intended for the Jekyll site.
