<h1 align="center">Welcome to nodecrawler 👋</h1>
<p>
  <img alt="Version" src="https://img.shields.io/npm/v/nodecrawler.svg">
  <a href="https://github.com/hp0844182/nodecrawler#readme">
    <img alt="Documentation" src="https://img.shields.io/badge/documentation-yes-brightgreen.svg" target="_blank" />
  </a>
  <a href="https://github.com/hp0844182/nodecrawler/graphs/commit-activity">
    <img alt="Maintenance" src="https://img.shields.io/badge/Maintained%3F-yes-green.svg" target="_blank" />
  </a>
  <a href="https://github.com/hp0844182/nodecrawler/blob/master/LICENSE">
    <img alt="License: ISC" src="https://img.shields.io/badge/License-ISC-yellow.svg" target="_blank" />
  </a>
</p>

> node 爬虫笔记

### 🏠 [Homepage](https://github.com/hp0844182/nodecrawler#readme)

最近同学让我帮做一个某电商的爬虫程序，作为一个前端，不会 pathon，只能想着用 node 写写看。

## 一、什么是爬虫？

    爬虫简而言之就是爬去网页上的信息。而网页结构就是一个树形结构，就像一个蜘蛛网一样。而爬虫程序就像一个蜘蛛，在这个蜘蛛网上去收取我们感兴趣的信息。

## 二、我使用 node 爬虫遇到的问题

1. 如何动态添加 url 队列对网站进行深度爬去。
2. 当你对大量的 url 进行爬取的时候，你的程序突然崩溃了，如何保证数据不丢失？
3. 如何向网页服务器隐藏你的请求源？

## 三、开始写爬虫前需要确定的两个东西。

1. Where to crawler? (要爬那的信息？)。
2. What to crawler? (你要爬什么信息？)。

> 工艺利其事必先利器

刚开始找了几个 node 爬虫库，但是效果不是很理想。不过皇天不负有心人，不过最终还是让我找到了一个：**Apify**.

## 四、使用 Apify 来开始我的爬虫之旅

### 1. 首先新建一个工程然后安装 apify 依赖。

```
npm i apify -S
```

> 接下来要确定一下爬取那个网站的信息（以豆瓣电影 Top 250 为例）

### 2.现在我们已经确定了要爬取的 url(https://movie.douban.com/top250),现在开始编写代码。

```
  // 引入apify
  const Apify = require('apify');
```

### 3.apify 提供了一个动态队列（requestQueue）来管理我们要爬取的 url,我们可以使用它来管理我们所有要爬取的 url。

```
  const Apify = require('apify');
  Apify.main(async ()=>{
    // 首先创建一个请求队列
    const requestQueue = await Apify.openRequestQueue();
    // 将要爬取的url添加到队列当中
    await requestQueue
  })
```
### 5.已经有了请求队列，接下来要做的是**What to crawler**。需要一个方法去解析请求的网页内容。

定义一个函数来解析网页内容,该函数之后会传入apify爬虫的一个实例当中
```
 const handlePageFunction = async ({request,$}) => {
   // 是不是对$很熟悉，其实就是node里的jquery
   // 先简单打印下网页的title.
   const title = $('title').text();
   console.log(`网页的title：${title}`);
 }
```
### 6.最后，创建一个CheerioCrawler 爬虫实例，并将requestQueue，handlePageFunction作为参数传入。然后启动爬虫
```
  const crawler = new Apify.CheerioCrawler({
        requestQueue,
        handlePageFunction
  })

  // 启动爬虫
  await crawler.run();
```
> 我们把代码做一下整合，然后启动爬虫。
```
  const Apify = require('apify');


  Apify.main(async () => {
      // 创建请求队列
      const requestQueue = await Apify.openRequestQueue();
      // 将要爬取的url添加到队列当中
      await requestQueue.addRequest({ url: 'https://movie.douban.com/top250' });

      const handlePageFunction = async ({ request, $ }) => {
          // 是不是对$很熟悉，其实就是node里的jquery
          // 先简单打印下网页的title.
          const title = $('title').text();
          console.log(`网页的title：${title}`);
      }

      //创建一个CheerioCrawler,将requestQueue，handlePageFunction作为参数传入
      const crawler = new Apify.CheerioCrawler({
          requestQueue,
          handlePageFunction
      })
      // 启动爬虫
      await crawler.run();
  })
```
> 运行代码，页面的标题成功被爬取

![title](https://s2.ax1x.com/2019/08/20/mGAI3Q.png)

> 到这里，就已经实现了一个简易的爬虫，但是还没有实现我们的需求（爬取完整的top250）。我们需要动态的去添加url,才能爬取到完整的250部电影。

### 8.获取所有要爬取的页面
> 初始url是首页，我们需要获取所有页码的页面，通过解析页面，我们可以通过以下方法获取当前页面的下一页。
```
  // 获取下一页的地址
  const link = $('.next a').attr('href');
```
> 但是我们发现获取的地址是一个相对地址，需要做一下转换。
```
  const {URL} = require('url');
  const Domain = 'https://movie.douban.com';
  const nextPage = new URL(link,Domain); 
```
> 接着将获取到的url动态添加到请求队列当中。

```
  requestQueue.addRequest({url:absoluteUrl});
```
### 9. 接下来需要修改一下handlePageFunction，来解析需要的电影信息。
```
/**
 * 解析网页，获取电影信息
 */
function parseMovie($) {
    const movieDoms = $('.grid_view .item');
    const movies = [];
    movieDoms.each((index, item) => {
        const movie = {
            rank: $(item).find('.pic em').text(), // 排名
            name: $(item).find('.title').text(), // 电影名
            score: $(item).find('.rating_num').text(), // 评分
            sketch: $(item).find('.inq').text() // 主题
        }
        movies.push(movie)
    })
    return movies
}
```
### 10. 再把代码整合到一起,运行看看结果
```
const Apify = require('apify');
const { URL } = require('url');

const Domain = 'https://movie.douban.com/top250';
Apify.main(async () => {
    // 创建请求队列
    const requestQueue = await Apify.openRequestQueue();
    // 将要爬取的url添加到队列当中
    await requestQueue.addRequest({ url: 'https://movie.douban.com/top250' });
    
    const handlePageFunction = async ({ request, $ }) => {
        // 解析页面获取电影信息
        const movies = parseMovie($);
        movies.forEach((item, i) => {
            console.log(`${item.rank}|${item.name}|${item.score}|${item.sketch}`)
        })
         // 获取下一页的地址
         const link = $('.next a').attr('href');
         if (link) {
             const nextPage = new URL(link, Domain);
             await requestQueue.addRequest({ url: nextPage.href })
         }
    }

    //创建一个CheerioCrawler,将requestQueue，handlePageFunction作为参数传入
    const crawler = new Apify.CheerioCrawler({
        // maxRequestsPerCrawl: 20, // <------ This is new too.
        requestQueue,
        handlePageFunction
    })
    // 启动爬虫
    await crawler.run();
})

/**
 * 解析网页，获取电影信息
 */
function parseMovie($) {
    const movieDoms = $('.grid_view .item');
    const movies = [];
    movieDoms.each((index, item) => {
        const movie = {
            rank: $(item).find('.pic em').text(), // 排名
            name: $(item).find('.title').text(), // 电影名
            score: $(item).find('.rating_num').text(), // 评分
            sketch: $(item).find('.inq').text() // 主题
        }
        movies.push(movie)
    })
    return movies
}

```
>⭐️运行下看看结果

![a](https://s2.ax1x.com/2019/08/20/mJinVf.png)

> 现在的运行结果已经满足我们的需求了，但是会不会觉得上面的代码有些麻烦，得找到链接、再转换、再添加到请求队列。可不可以给出一个url规则，然后程序自动帮我添加到队列中呢？

### 11. 使用Apify.utils.enqueueLinks()来自动添加需要爬取的页面
  首先引入 enqueueLinks
  ```
  const {
    utils: { enqueueLinks },
} = Apify;
  ```
  去掉之前url提取的代码，然后添加以下代码
```
   await enqueueLinks({
            $,
            requestQueue, 
            selector: 'div.item > a', // 跳转的a标签
            baseUrl: request.loadedUrl, //根据baseUrl会将a中的href补全
        });
```
## 结尾
  到这里，一个简单的爬虫程序算是写完了，但是这里还少了ip代理以及请求源伪装。后面再加上
## Install

```sh
npm install
```

## Run tests

```sh
npm run test
```

## Author

👤 **HuangPeng**

- Github: [@hp0844182](https://github.com/hp0844182)

## 🤝 Contributing

Contributions, issues and feature requests are welcome!<br />Feel free to check [issues page](https://github.com/hp0844182/nodecrawler/issues).

## Show your support

Give a ⭐️ if this project helped you!

## 📝 License

Copyright © 2019 [HuangPeng](https://github.com/hp0844182).<br />
This project is [ISC](https://github.com/hp0844182/nodecrawler/blob/master/LICENSE) licensed.

---

_This README was generated with ❤️ by [readme-md-generator](https://github.com/kefranabg/readme-md-generator)_
