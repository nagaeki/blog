---
title: "Why I'm Transitioning to Static Websites"
date: 2023-02-12T22:24:17+08:00
description: "Moving from Wordpress to Hugo"
tags: [web]
featured_image: "featured_image.webp"
draft: false
hidden: false
---

```
Note: This does not serve as a guide. It is only a reminder in case I forget this again.
```

What's a static website? What are advantages of static websites? Please check [this blog from Cloudflare](https://www.cloudflare.com/learning/performance/static-site-generator/).

I'm not in a position to say how one option is correct and the other option is not. This is only a log for myself.

# Initial choice: WordPress

The reasons for which I chose to use WordPress in the beginning is simple:
- Easy to follow tutorials
- Beginner-friendly editor
- Not too bad-looking pages
- Abundant choice of themes/plugins

But after using WordPress for a while, I also realized some of its weaknesses:
- Performance is much worse than static pages
- CVEs appear often
- I don't need most of the functions a dynamic site provides
- The editor can be hard to use sometimes

# Turning to Hugo

There are a lot of choices for static site generators. I chose Hugo for the following options:
- Good performance
- Great plugins/themes/documentation
- The fact that it uses Markdown for editing

At the same time, I use VSCode and Git for writing and source management, and overall have a wonderful experience.

Although initially I did not have much interest for solutions that require me to use the command line and external editors, after about half an hour of reading the documentation, it turned out to be not too difficult.

I also plan to remake the DN42 pages and other websites to be static where possible, to have a better editing experience.

# Common knowledge and commands for Hugo

Source of truth for this part comes from the [Hugo Docs](https://gohugo.io/getting-started/quick-start/)

## Quickstart

```
hugo new project quickstart
```

This command asks hugo to create a folder called quickstart, and generates a project skeleton within that folder. The project skeleton looks like this.

```
my-project/
├── archetypes/
│   └── default.md
├── assets/
├── content/
├── data/
├── i18n/
├── layouts/
├── static/
├── themes/
└── hugo.toml         <-- project configuration
```

Depending on your site's requirements, you might not need some of those folders. After the site is built, the folders `public/` and `resources/` are also creates as the built files.

Each of the folders contribute to the website's content or behaviour. For what each folder is used, refer to this [docs](https://gohugo.io/getting-started/directory-structure/)

## Page bundles

Hugo uses a folder structure that bundles images and other resources into [**Page bundles**](https://gohugo.io/content-management/organization/). Page bundles are content folders that holds `index.md` or `_index.md` at their foot. The resources within page bundles are called page resources, and they are only available to the page with which they are bundled.

Content should be organized in a manner that reflects the rendered website. For example, the following structure should work without additional configuration.

```
.
└── content
    └── about
    |   └── index.md  // <- https://example.org/about/
    ├── posts
    |   ├── firstpost.md   // <- https://example.org/posts/firstpost/
    |   ├── happy
    |   |   └── ness.md  // <- https://example.org/posts/happy/ness/
    |   └── secondpost.md  // <- https://example.org/posts/secondpost/
    └── quote
        ├── first.md       // <- https://example.org/quote/first/
        └── second.md      // <- https://example.org/quote/second/
```

A more detailed example with page resources could look like this.

```
content
└── post
    ├── first-post
    │   ├── images
    │   │   ├── a.jpg
    │   │   ├── b.jpg
    │   │   └── c.jpg
    │   ├── index.md (root of page bundle)
    │   ├── latest.html
    │   ├── manual.json
    │   ├── notice.md
    │   ├── office.mp3
    │   ├── pocket.mp4
    │   ├── rating.pdf
    │   └── safety.txt
    └── second-post
        └── index.md (root of page bundle)
```

Note that the page resources of page `first-post` are not visible to page `second-post`.

```
content/
├── blog/
│   ├── hugo-is-cool/
│   │   ├── images/
│   │   │   ├── funnier-cat.jpg
│   │   │   └── funny-cat.jpg
│   │   ├── cats-info.md
│   │   └── index.md
│   ├── posts/
│   │   ├── post1.md
│   │   └── post2.md
│   ├── 1-landscape.jpg
│   ├── 2-sunset.jpg
│   ├── _index.md
│   ├── content-1.md
│   └── content-2.md
├── 1-logo.png
└── _index.md
```

This example shows nested page bundles, namely `content/_index.md`, `content/blog/_index.md`, and `content/blog/hugo-is-cool/index.md`. Note that the home page cannot contain other content pages, although other files such as images are allowed.

Hugo only makes sure each content is rendered to the target paths, however the navigation to the contents at each path are up to the website's author to implement, as well as the theme to decide on its apperance. For example, a blog based theme could list all of its posts at the home page, while a company facing theme probably won't make each product page visible directly on the root page. Therefore it is important to refer to the theme's manual and make changes accordingly.

## Initializing Git

```
cd quickstart
git init
git submodule add https://github.com/gohugo-ananke/ananke themes/ananke
```

Of course, you would use git for source control on your content. Here we can add a theme as a Git submodule, to the path `themes/ananke`. More about this will be featured on the part, "How to use Git submodules".

## Editing and viewing content

To add a new page to the project's content, you would use the following command.

```
hugo new content content/posts/my-first-post.md
```

Hugo will create the file to the path do designated, from the [archetype](https://gohugo.io/content-management/archetypes/) you chose, which is in this instance `posts`.

By default the page's front matter will set the value of `draft` to `true`, which will not be published by default.

To start Hugo's development server, you would run the following command.

```
hugo server
```

However, to include draft content, you would run the following command.

```
hugo server --buildDrafts
hugo server -D
```
## Publishing the project

When you choose to publish your project, Hugo renders all build artifacts to the `public` directory in the root of your project. This includes the HTML files for every site, along with assets such as images, CSS, and JavaScript. The command to do so is very simple.

```
hugo
```

It might be important to remember that Hugo does not clear the public directory before building your project. Existing files are overwritten, but not deleted. This behavior is intentional to prevent the inadvertent removal of files that you may have added to the public directory after the build. Depending on your needs, you may wish to manually clear the contents of the public directory before every build.

Same goes with the resources folder, which is said to `By default this cache directory includes CSS and images. Hugo recreates this directory and its content as needed`. Therefore in some cases it might be benefitial to remove that folder by hand.


# How to use Git submodules

Source of truth to this section should be on the [Git handbook](https://git-scm.com/book/en/v2/Git-Tools-Submodules)

## What are submodules?

Sometimes you need to refer to another repository from your own. However, if you simple refer to it by code by pointing to another directory outside of the current repository, you lose the advantages of using Git for version control on that, making updating it when newer upstream sources are available extremely difficult. Therefore, normally it is preferable to hold the code of that repository, within the same Git version controlled environment.

There are a few ways to do that. For example, some programming languages might have builtin package managers that allow importing packages. However, as is the case with Hugo themes, sometimes we do need the raw code at our hands. There is also the problem with simply including the library, which is that it is difficult to customize the library in anyway, or to make sure each time that the library is installed on a target machine.

Git addresses this problem with submodules, which allows the user to keep another Git repository as a subdirectory of a Git repository, while also keeping the commits separate.

Other advantages of using Git submodules also include:

- Code safety when an external component is changing too fast, or when upcoming changes will break your project.
- When an external component doesn't change too often, and you want to track it as a dependency.

## Hugo modules

Hugo allows the use of external resources as modules through its builtin management feature.

```
A module is a packaged combination of components which may contain archetypes, assets, content, data, templates, translation tables, and static files. A module may be a theme, a complete project, or a smaller collection of one or more components.
```

Source: [Hugo docs](https://gohugo.io/hugo-modules/use-modules/)

To put it simply, Hugo provides a way to manage external resources used by Hugo through the `hugo mod` command, which allows inclusion of external files through a subdirectory in the project folder, a external path on the system's mountpoint, or even just a link to a Git repository. Importing, updating, caching, removing and other actions are all controlled by Hugo.

I do believe that this is a great tool to get started when other resources are necessary. However, since that this is a function confined to Hugo, the option to use the submodule function of Git, which could also be used in other projects unrelated to Hugo, could be a better option.

## Quirks of Git submodules

Although submodules are represented by just another directory on your filesystem, Git sees it as a submodule and doesn't track its contents when you're not in that directory. Instead, it sees the directory as a whole, as a certain commit.

Therefore, when you wish to update contents of the submodule, it might not be as straightforward as editing a simple file, as there is a flow that needs to be followed.

- Go to the submodule's directory
- Edit the submodule
- Commit changes to the submodule
- Push the submodule's changes to remote
- Now the parent repository sees a new version to the submodule
- Commit changes to the parent repository, and pushto remote

It is important that the editor does not forget to push changes to the submodule to remote, before the parent repository is pushed. This is because Git only tracks the submodule to a commit, and when the parent repository is pushed without the new version of the submodule being available, other users trying to use the repository will fail to do so.

## Common commands to use Git submodules

Again, the source of truth is the Git handbook, linked above. This part will only cover some common commands that I will probably use often.

### Adding a submodule

To start to use a submodule, you would use the following command.

```
git submodule add https://link-to-repo
```

This will result in the repository being cloned, and the `.gitmodules` file created.

```
[submodule "submodule-a"]
    path = submodule-a
    url = https://link-to-repo
```

Note that if you're using code from another developer, it might be a good idea to fork that repository first. This is because while using submodules, you probably with to edit the files to a certain point, however using the original developer's repository directly does not allow your changes to be pushed to remote, at least the changes that only apply to yourself.

### Cloning a repository with submodules

When cloning a project with a submodule in it, by default the directories that contain the submodules will be created, but not the contents inside those directories. To also clones the submodules, you need to run these commands.

```
git submodule init # inits local configuration
git submodule update # fetches data from remote
```

Another way to do this is to pass `--recursive-submodules` to the `git clone` command. This has the advantage of being able to recursively fetch data if any of the submodules in your repository has submodules of themselves.

```
git clone --recursive-submodules https://link-to-repo
```

Finally, if you have already cloned the repository but forgot to use `--recursive-submodules`, you can combine `git submodule init` and `git submodule update` with the following one liner.

```
git submodule update --init
git submodule update --init --recursive # if submodules are recursive
```

### Updating upstream changes from submodule remote

One common way to use submodules is to consume its content, but not make any modifications on your own. Thus, it might be necessary to update the submodule from its upstream from time to time.

To check for new work in a submodule, go into its directory and run these commands.

```
git fetch
git merge origin/master
```

If you commit at this point then you will lock the submodule into having the new code when other people update.

An easier way to do this exists, if you do not wish to fetch and merge in the subdirectory by hand.

```
git submodule update --remote
git submodule update --remote submodule_name # when only updating one submodule
```
