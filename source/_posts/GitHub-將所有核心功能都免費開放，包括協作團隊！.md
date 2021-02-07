---
title: GitHub 將所有核心功能都免費開放，包括協作團隊！
date: 2020-04-16 09:10:59
tags:
- github
- gitlab
---

小編好久沒發文了 Orz，來發個近日資訊業最大條新聞跟個人看法

---

GitHub 被微軟買走之後做了蠻多事情，像是前幾個月 npm 被 GitHub 買下來，但這其實也不意外，因為這兩年 GitHub 在 JavaScript 的支援度就愈來愈高，不止可以在網頁上面做 audit，也可以做 navigate，功能蠻齊全的。被買下來之後，之後可以期待在 GitHub 上面直接 deploy npm 模組了。

另外一個就是這次的 private repo 完全免費了，小編記得之前 private repo 免費好像有些限制，像是只限 5 人的樣子，但現在可說是完全不限，有富爸爸果然是不一樣！

---

再來說一下個人選擇，如果是可以給一般人看的內容，小編一定是用 GitHub，畢竟資訊人誰沒有一個 GitHub 帳號呢？然後個人接案因為不能公開原始碼，所以當然放在 GitLab (以前是 Bitbucket)。現在 Funliday 的 git repo 因為原始碼不能公開，當然也是放在 GitLab。

然後因為 Funliday server 是放在 Heroku 上面，Heroku 上面有個功能可以直接結合 GitHub 的 push 之後 deploy，但因為 Funliday 用的是 GitLab，所以沒辦法直接用這功能，只能繞個彎改用 dpl，把 GitLab 整合進 Heroku 裡。

但也因為用了 GitLab 之後，才知道它的功能比 GitHub 多太多，對一般公司所需要的權限控制，小編是覺得比 GitHub 完整。然後對於 CI/CD 的支援度也很強大，現在還支援 build docker image 的功能。

---

不過當 GitHub private 免費之後，對於沒有依賴 GitLab 權限控制或 container 功能的使用者，還蠻有可能移回 GitHub 的，畢竟能在同一個 hosting 操作當然比較方便。所以以後小編開 repo 的考量，可能就不再是 private 或 public 了，而是在於 deploy 的方便性，或是能不能 build 出 artifact 了。

* GitLab 自評與 GitHub 的差異: https://about.gitlab.com/blog/2020/04/14/github-free-for-teams/index.html