# frozen_string_literal: true

source "https://rubygems.org"

gem "jekyll", ENV["JEKYLL_VERSION"] if ENV["JEKYLL_VERSION"]
gem "kramdown-parser-gfm" if ENV["JEKYLL_VERSION"] == "~> 3.9"

group :jekyll_plugins do
  gem 'jekyll-katex'
end

gem "jekyll-remote-theme"

gem "github-pages", group: :jekyll_plugins
