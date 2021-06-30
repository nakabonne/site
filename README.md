# site
My personal site built with [hugo](https://github.com/gohugoio/hugo).

## Dependencies

Hugo:

```console
❯ hugo version
hugo v0.84.3+extended darwin/amd64 BuildDate=unknown
```

**Note** that the generated files may make a difference if the version is even slightly different.

## How to start blogging
- Add new file to `content/posts`
- Add an image to `static/img` if needed.

## Publish

Be sure to build with `hugo` command before pushing.
Push it and then two actions are triggered.

- Push the image based on nginx to Docker Hub
- Deploy to Firebase Hosting
