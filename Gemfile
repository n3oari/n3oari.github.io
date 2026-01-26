# frozen_string_literal: true

source "https://rubygems.org"

# Usamos la versión de GitHub para saltar restricciones de versión de Ruby
gem "jekyll-theme-chirpy", github: "cotes2020/jekyll-theme-chirpy", branch: "master"

gem "jekyll", ">= 4.3.0"
gem "webrick", "~> 1.8"

group :test do
  gem "html-proofer", "~> 5.0"
end

platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

gem "wdm", "~> 0.2.0", :platforms => [:mingw, :x64_mingw, :mswin]