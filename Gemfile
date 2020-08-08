source "https://rubygems.org"

gem "jekyll"
gem "minimal-mistakes-jekyll"

group :jekyll_plugins do
    gem "jekyll-feed"
    gem "jekyll-seo-tag"
    gem "jekyll-sitemap"
    gem "jekyll-paginate-v2"
    gem "jekyll-include-cache"
    gem "jemoji"
    gem "jekyll-algolia"
    gem "jekyll-archives", git: "https://github.com/jekyll/jekyll-archives.git", branch: "master"
  end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]
