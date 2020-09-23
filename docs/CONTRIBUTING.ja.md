# blog (minimal-mistakes-jekyll) のビルド方法 

Ruby, Gem, Bundler などはセットアップ済とする

## nokogiri 対応
plugin:jemoji 関係
- deb 系のみ  
```sudo apt install -y libxslt-dev build-essential ruby-dev zlib1g-dev liblzma-dev libxmlsec1-dev libxml2-dev pkg-config```

- windows ruby系のみ  
```ridk exec pacman -S mingw-w64-x86_64-libxslt```

- 共通  
```bundle config build.nokogiri --use-system-libraries```  
```gem install nokogiri --platform=ruby -- --use-system-libraries```

## bundle install 
4-5 分かかる場合がある  
```bundle install --jobs=4```

## jekyll server 起動
```bundle exec  jekyll serve --trace```


# 記事を書く人向け



# ブログシステム (Jekyll + minimal-mistakes theme) を変更する人向け



