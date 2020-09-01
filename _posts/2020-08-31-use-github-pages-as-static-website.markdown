---
layout: post
title:  "Use GitHub Pages as Static WebSite"
date:   2020-08-31 14:21:47 +1000
categories: GitHub 
tags:
- GitHub
- Documentation
---

[GitHub Pages](https://pages.github.com/) makes it easy to create static webpages, (no database or server). Ideal for sites with little change to its information or information that can be automatically generated. For example as part of a build process, creating documentation.

This post will go through how you setup a user blog site, like this one, in a few steps. You will maintain the site locally and push the changes to GitHub to publish the information on your site.

As GitHub Pages is powered by [Jekyll](https://jekyllrb.com/) we need Ruby installed locally so we can maintain the site.

### Install Jekyll and Ruby

For windows you can use the [RubyInstaller](https://jekyllrb.com/docs/installation/windows/) or [chocolatey](https://chocolatey.org/) as follows:

```
choco install ruby -y   
refreshenv  
gem install bundler   
```

Either way, just accept the default configuration. 

Check if Ruby is installed by using the command:

```
ruby -v
```   
This should return the version number. 

Similarly, you can check if RubyGems is installed by typing:

```
gem -v
```

If both are installed then install Jekyll:

```
sudo gem install jekyll
```

Once completed you can verify the Jekyll installation with the command:

```
jekyll -v
```


### Create a Repository for the Site
Head over to GitHub and create a new repository named username.github.io, where username is your username (or organization name) on GitHub.

In my case this repository would look like this as my username is bengthedberg:

![build](\assets\github_repo.png)

Once it has been created you can review the settings:

![build](\assets\github_page_settings.png)



### Create the Site

There are different ways to create the actual site:

- Clone an existing free Jekyll theme using any of these [themes](https://pages.github.com/themes/)
- Pay for an Jekyll Theme, using a site like [this](https://jekyllthemes.io/github-pages-themes)
- Create a new, default Jekyll site with the command:`jekyll new <site name>`

In this example we are just going to create a new, default site:

```
jekyll new MyBlog
```
This should create a folder called MyBlog. 

Make this your current folder and modify the `_config.yml` file. The properties in the file is mostly self explanatory, but with the minima theme there are some addition settings, see this [page](https://github.com/jekyll/minima/blob/v2.5.0/README.md) for more customisations.


Once you are ready you can run the following command to test your site locally.

```
jekyll server --watch
```

This will start the site locally, normally on [http://localhost:4000/](http://localhost:4000/)

### Update the Site 

Once you are ready you can upload the site to GitHub:

First initialise git locally and link it to your github repository: 

```
git init
git remote add origin https://github.com/<user name>/<user name>.github.io
```

Then push all content:

```
git add .
git commit -m "Initial commit"
git push origin master
```

Your site is now live, go to your GitHub Page link and check it out. Note that initially it can take up to 20 minutes for it to work so be patient.

You can verify the last time the site was rebuilt by clicking the github-pages link in the github repository:

![build](\assets\github_environment.png)

This page will display the latest github deployment as well as an activity log.

![build](\assets\github_activity.png)


> Note: Any build failure will be emailed so if your github site is not updated you should check your email that is registered to your github account.  

### Posts

You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. You can rebuild the site in many different ways, but the most common way is to run `jekyll serve`, which launches a web server and auto-regenerates your site when a file is updated.

Jekyll requires blog post files to be named according to the following format:

`YEAR-MONTH-DAY-title.MARKUP`

Where `YEAR` is a four-digit number, `MONTH` and `DAY` are both two-digit numbers, and `MARKUP` is the file extension representing the format used in the file. After that, include the necessary front matter. Take a look at the source for this post to get an idea about how it works.

Jekyll also offers powerful support for code snippets:

{% highlight ruby %}
def print_hi(name)
  puts "Hi, #{name}"
end
print_hi('Tom')
#=> prints 'Hi, Tom' to STDOUT.
{% endhighlight %}

### Further Information

Check out the [Jekyll docs][jekyll-docs] for more info on how to get the most out of Jekyll. File all bugs/feature requests at [Jekyll’s GitHub repo][jekyll-gh]. If you have questions, you can ask them on [Jekyll Talk][jekyll-talk].

[jekyll-docs]: https://jekyllrb.com/docs/home
[jekyll-gh]:   https://github.com/jekyll/jekyll
[jekyll-talk]: https://talk.jekyllrb.com/
