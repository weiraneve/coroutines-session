# Tech-WIKI
A coroutines-session tech docs by jekyll.

## System Requirements
- Ruby 2.6.5+
- Bundler 2.3.19+
- Node.js 16.17.0+

## Setup(Mac)
- Install ruby, Node.js and bundler
```sh
brew install ruby
gem install bundler
brew install node
```

## View Docs
- Start local server  
```sh
bundle exec jekyll serve
```
- View the docs
```sh
open http://localhost:4000
```

## Edit Docs
- Follow this document style
[Document Style Guide](https://github.com/ruanyf/document-style-guide)

### Check lint issue
```
npm run check
```

### Autofix lint issue
```
npm run format
```

### Disable textlint when meet error
```
<!-- textlint-disable -->你要忽略的文本<!-- textlint-enable -->
```