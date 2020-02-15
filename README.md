# Relaynet Protocol Suite Specifications

This is a [Jekyll](https://jekyllrb.com/) site for all the official [Relaynet](https://relaynet.link/) specifications. A live version of the site is available at [specs.relaynet.link](https://specs.relaynet.link).

## Run locally

If it's the first time you're going to generate the documentation, start by checking that [Ruby](https://www.ruby-lang.org/en/downloads/) and [Bundler](https://rubygems.org/gems/bundler) (`bundle`) are installed and available in your `$PATH`. Then install the Ruby dependencies for this project, including Jekyll:

```bash
bundle install --path vendor/bundle
```

You're now ready to serve the site locally:

1. Start Jekyll:
   ```bash
   bundle exec jekyll serve
   ```
1. Go to [http://localhost:4000](http://localhost:4000).
