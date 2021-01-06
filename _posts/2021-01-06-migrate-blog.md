---
layout: post
title: 'Migrate my blog from WordPress to Jekyll'
---

## Why Jekyll?

  My blog was on WordPress hosted by Godaddy. Using WordPress is simple for non-technical people, they don't have to understand HTML and CSS.

  Jekyll is the engine behind GitHub, which you can use to host sites right from your GitHub repositories. As a file-based CMS, it can render Markdown and Liquid templates. And it's <strong>free</strong>.

  As an IT guy, similar to "infrastructure as code", I can handle "post as code" and merge my posts into GitHub branch.

## Before Migration

  * Install Jekyll - [https://jekyllrb.com/docs/installation/](https://jekyllrb.com/docs/installation/)
  * Choose a theme, or design your own styles
  
    I forked [Hyde](https://github.com/poole/hyde) to my local.

## Fix issue in Big Sur (MacOS)

  After upgrading my Mac to the latest OS, Jekyll stopped working and threw this error:

  ```
  Could not find eventmachine-1.2.7 in any of the sources. Run `bundle install` to install missing gems.
  ```

  Then Google gave me the following solution:

  * Install Xcode 12.3
  * Install Command Line Tools for Xcode 12.3
  * Install rbenv

      ```shell
      git clone https://github.com/rbenv/rbenv.git ~/.rbenv
      cd ~/.rbenv && src/configure && make -C src
      # Add ~/.rbenv/bin to your $PATH
      echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bash_profile
      xcode-select --switch /Applications/Xcode.app/Contents/Developer
      # Navigate to your Jekyll root
      bundle install
      ```

      Then Jekyll worked on my local

      ```
      jekyll serve
      ```

## Data Migration

  According to [https://import.jekyllrb.com/docs/wordpress/](https://import.jekyllrb.com/docs/wordpress/), I need to install <strong>jekyll-import</strong> to import my posts from WordPress.

  1. Install all the required plugins
  
      * Install sequel

          ```shell
          sudo gem install sequel
          ```

      * Install unidecode

          ```shell
          sudo gem install unidecode
          ```

      * Install jekyll-import

          ```shell
          sudo gem install jekyll-import
          ```


  2. Run jekyll-import

      ```ruby
      ruby -r rubygems -e 'require "jekyll-import";
        JekyllImport::Importers::WordPress.run({
          "dbname"         => "[mysql_db]",
          "user"           => "[mysql_user]",
          "password"       => "[mysql_pwd]",
          "host"           => "[mysql_host]",
          "port"           => "3309",
          "socket"         => "",
          "table_prefix"   => "wp_[my_table_prefix]",
          "site_prefix"    => "",
          "clean_entities" => true,
          "comments"       => true,
          "categories"     => true,
          "tags"           => true,
          "more_excerpt"   => true,
          "more_anchor"    => true,
          "extension"      => "html",
          "status"         => ["publish"]
        })'
      ```

      Follow [Open phpMyAdmin](https://au.godaddy.com/help/open-phpmyadmin-24573) to get those info:

        * [mysql_db] - the same as mysql_user for my instance
        * [mysql_user]
        * [mysql_pwd]
        * [mysql_host]
        * [my_table_prefix] - an extra prefix after wp_
      
      I encountered an error when I ran the above command.

      ```
      LoadError: cannot load such file -- mysql2
      ```

      As I didn't have MySql installed on my local environment.
    
      ```shell
      brew install mysql
      ```

      ```shell
      sudo gem install mysql2 --platform=ruby
      ```
  
  3. Convert html to md

      Once import job complete, all of my previous posts were downloaded as html files, I converted them to MarkDown format and then manually fixed the image references.

  4. Google Tag Manager

      "jekyll-google-tag-manager" is not supported by GitHub as it's not in [Dependency Version](https://pages.github.com/versions/). Therefore, I have to include GTM html into head and body.

  5. Other useful plugins 
      
      * Sitemap - jekyll-sitemap
      * 301 Redirect - jekyll-redirect-from
        
        In the post md file, I just need to include <strong>redirect_from</strong> in yaml header e.g. 

          ```
          ---
          layout: post
          title: 'AZ-400: Designing and Implementing Microsoft DevOps Solutions'
          redirect_from:
            - /az-400/
          ---
          ...MY POST CONTENT...

          ```


