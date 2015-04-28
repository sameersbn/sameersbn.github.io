# Octopress Cheatsheet

## Requirements

```bash
apt-get install nodejs ruby-dev
gem install --no-ri --no-rdoc bundler
```

## Install Gems

```bash
bundle install --path vendor/bundle
```

## Install Theme

```bash
bundle exec rake install['octohead']
```

## Blogging

```bash
bundle exec rake generate   # Generates posts and pages into the public directory
bundle exec rake watch      # Watches source/ and sass/ for changes and regenerates
bundle exec rake preview    # Watches, and mounts a webserver at http://localhost:4000
```

## New Posts

```bash
bundle exec rake new_post["title"]
```

### Example

```bash
bundle exec rake new_post["Zombie Ninjas Attack: A survivor's retrospective"]
# Creates source/_posts/2011-07-03-zombie-ninjas-attack-a-survivors-retrospective.markdown
```

## New Pages

```bash
bundle exec rake new_page[super-awesome]
# creates /source/super-awesome/index.markdown

bundle exec rake new_page[super-awesome/page.html]
# creates /source/super-awesome/page.html
```

## Deploy to Github

```bash
bundle exec rake deploy
```
