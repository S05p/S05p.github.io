source "https://rubygems.org"

# Jekyll 버전
gem "jekyll", "~> 4.3.2"

# Ruby 3.4+ 호환성을 위한 필수 gem들
gem "csv"
gem "logger"
gem "ostruct"
gem "base64"

# Jekyll 플러그인
group :jekyll_plugins do
  gem "jekyll-feed"
  gem "jekyll-sitemap" 
  gem "jekyll-paginate-v2"
end

# Windows 및 JRuby 지원
platforms :mingw, :x64_mingw, :mswin, :jruby do
  gem "tzinfo", ">= 1", "< 3"
  gem "tzinfo-data"
end

# Windows에서 디렉토리 감시 성능 향상
gem "wdm", "~> 0.1.1", :platforms => [:mingw, :x64_mingw, :mswin]

# 추가 유틸리티 (선택사항)
gem "webrick", "~> 1.7" # Ruby 3.0+ 에서 필요할 수 있음