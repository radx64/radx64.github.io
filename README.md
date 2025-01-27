# radx64.github.io 


### Notes

#### Current theme

[https://github.com/cotes2020/chirpy-starter](https://github.com/cotes2020/chirpy-starter)



#### Installing ruby

```
sudo apt install ruby-dev ruby-bundler
```

#### Installing gems

To install ruby gems as `non-root` user (not needed as Gemfile has all the gems)

Add to .bashrc
```
export PATH="$PATH:$HOME/.gems/bin"
export GEM_HOME="$HOME/.gems"
```

and then use install like this

```
gem install <gems_name> --install-dir $HOME/.gems
```
or simpler
```
gem install <gems_name>
```

#### Changes in `Gemfile`

After modification of `Gemfile` run
```
bundle install
```

#### Local site testing

Run
```
bundle exec jekyll serve
```
