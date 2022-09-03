# frozen_string_literal: true

source "https://rubygems.org"

gemspec

group :test do
  gem "html-proofer", "~> 3.1"
end

# Windows and JRuby does not include zoneinfo files, so bundle the tzinfo-data gem
# and associated library.
install_if -> { RUBY_PLATFORM =~ %r!mingw|mswin|java! } do
  gem "tzinfo", "~> 1.2"
  gem "tzinfo-data"
end

# Performance-booster for watching directories on Windows
gem "wdm", "~> 0.1.1", :install_if => Gem.win_platform?

# Jekyll <= 4.2.0 compatibility with Ruby 3.0
gem "webrick", "~> 1.7"
gem 'jekyll-seo-tag', '~> 2.8'
gem 'jekyll-archives', '~> 2.2', '>= 2.2.1'
gem 'jekyll-sitemap', '~> 1.4'
gem 'parallel', '~> 1.22', '>= 1.22.1'
gem 'rainbow', '~> 3.1', '>= 3.1.1'
gem 'typhoeus', '~> 1.4'
gem 'yell', '~> 2.2', '>= 2.2.2'
gem 'zeitwerk', '~> 2.6'
gem 'ethon', '~> 0.15.0'
gem 'etc', '~> 1.1'
