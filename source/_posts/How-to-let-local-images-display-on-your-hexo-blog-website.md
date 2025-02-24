---
title: How to let local images display on your hexo blog website
date: 2020-11-27 14:30:32
categories: [IT]
tags: [hexo, blog, online image, config, Node.js, router]
toc: true
cover: /img/hexoribbon.jpg
thumbnail: /img/hexoicon.png
---

Hexo blog framework friendly supports markdown which is the most popular syntax for technical blog writing. When you try to illustrate some ideas, it's common to use screenshot photos in your articles to get reader better understanding so that you have to insert images into your blog, unlike typing text content that is straight forward, inserting images and deploying them to public blog site might be some tricks there, let's see some common scenarios on images display issue and how to resolved them.

## Use absolute path to insert image

The intuitive way for junior blogger is referencing absolute path + file name, I would say there is no any problem if the blog is only for local review or your local machine is a web server to host your blog website, otherwise, your images won't be displayed on your public blog site after deployment if without any configuration. Like below shows when you use that way inert on your local blog, images can be displayed as expected

<!-- more -->

[![absolutePath_image.png](/img/screenshots/absolutePath_image.png)

but it will cause issue on display when you deploy your article to public blog website like below shown

[![absolutePath_renderIssue.png](/img/screenshots/absolutePath_renderIssue.png)

## Use relative path based on _config.yml file 

Actually, hexo blog is powered by Node.js in the backend, its general configuration file _config.yml shows the way to properly allocate your resources including image, css, font and js etc. By default, your images should be put into `blog/themes/your theme name/source/images` , when you reference it, use below syntax no matter you use Windows or Linux,

```markdown
![your image's name](/images/your_image_file_name.png)
```

now, you can push your blog to public blog site and your images can be presented there well

![image_showOnWeb.png](/img/screenshots/image_showOnWeb.png)

but the problem is you lose your sight on local but only are able to see on blog website after you deployment, that is not you really want like below shown

[![image_noShow_local.png](/img/screenshots/image_noShow_local.png)

## Use online image management tool 

To be able to solve image display problem on both local and public site, the best way is utilize online image management website to upload your images then it will generate corresponding URLs for those photos, you can replace path name of your local images by URL. Now your image is going to be rendered by both your local markdown editor and web browser.

[![image_showOnLocal1.png](/img/screenshots/image_showOnLocal1.png)

[![image_showOnLocal2.png](/img/screenshots/image_showOnLocal2.png)

[![image_showOnWeb2.png](/img/screenshots/image_showOnWeb2.png)

Another advantage of managing your photos by online image tool is your images are in the cloud with permanent URLs so that you won't lose them even something bad happened on your local machine.

I use [wailian.work](https://www.wailian.work/) to manage my images, it's free and provide markdown link for each of your uploaded picture.

## A small thinking about Node.js top route design and file render  

The root cause of confusion here might be the Node.js route design, unlike Apache and Nginx web server which come with web container so that the route can be your local folder path name and render your local html file in the folder, in this case, you do tell the file local path from URL address, but for Node.js, it doesn't have web container, therefore, we have to design route for local web files, in other words, you are not able to tell the local file path by URL address for Node.js web server. Let's see an example

```javascript
const fs = require('fs');
const path = require('path');
const url = require('url');

exports.static = function (request, response, staticPath) {
    let pathname = url.parse(request.url).pathname;
    pathname = pathname == '/' ? '/index.html' : pathname;
    let extname = path.extname(pathname);
    if (pathname != '/favicon.ico') {
        fs.readFile('./' + staticPath + pathname, async (err, data) => { 
            if (!err) {
                let mime = await getFileMime(extname.split('.')[1]);
                response.writeHead(200, { 'Content-Type': `${mime}; charset="utf-8"` });
                response.end(data);
            }
        })
    }
}
```

Assume user request images, the URL is probably like https://yourdomainname/images, but in the backend, in fact the Node.js read file from path `'./' + staticPath + pathname` such as `/root/web_project1/static/images`. URL address doesn't reflect file path, they are relatively independent in Node.js, by knowing that, we are going to dig a little deeper to take a look at _config.yml file and come up with idea why browser is able to render out to image file but text editor can't. Open _config.yml file 

```bash
vim ./blog/_config.yml
```

```yaml
# URL
## If your site is put in a subdirectory, set url as 'http://example.com/child' and root as '/child'
url: https://hermanteng19.github.io
root: /
permalink: :year/:month/:day/:title/
```

here list the route on your blog site which should be `https://hermanteng19.github.io/` as home page and corresponding html file should be put into "/" directory, what about images file? It is going to be "/images", so far, we know the directory in below actually is route or web request URL, that is why text editor is not able to find the real file path.

```markdown
![mysql-connection](/images/mysql_conn_established.png)
```

But, you might still confuse about why browser can render the image file, now let's continue to see what's going on web server. Enter your local hexo blog path

```bash
ls -l ./blog
```

[![root_public_folder.png](/img/screenshots/root_public_folder.png)

you can find there is `public/` folder which is your blog website "/root" folder, all web files such as html, css, js, image, font are in there, let's enter it and take a look

[![root_public_images_folder.png](/img/screenshots/root_public_images_folder.png)

`images/` folder is on the list, continue to drill down, `mysql_conn_established.png` file is on the list. So now you may be clear by Node.js route design, when web browser visits images on your blog website, it actually read the files from "/public/images/", not from your local file absolute path like "themes/your_theme_name/source/images".

It does cause some confusion by this special type of route design, but in the other way, I would say it's quite smart, you can design out a very nice, neat and beautiful URL for your website, which would bring very good user experience and impress website visitors. Think about no matter how ugly your local file path is like `/root/jfdioaw/_jfidosjo_jfi123/oneMoreFolder/13u8030/product_1.html`, but your website URL always looks like `http://yourdomainname/business/product/product_1.html`.

At last, I want to thank [CodeSheep](https://www.codesheep.cn/) for help me with the idea of leveraging online photo mangement tool to solve this problem!








