---
title: 控制器push过程中的生命周期
categories: iOS
tags: 基础知识
date: 2016-12-05 09:38:46

---

# A 展现的时候

- A viewDidLoad
- A viewWillAppear
- A viewWillLayoutSubviews
- A viewDidLayoutSubviews
- A viewDidAppear

# A push 到 B 的时候

- A viewWillDisappear
- B viewDidLoad
- B viewWillAppear
- B viewWillLayoutSubviews
- B viewDidLayoutSubviews
- A viewDidDisappear
- B viewDidAppear

# B pop 回 A 的时候

- B viewWillDisappear
- A viewWillAppear
- B viewDidDisappear
- A viewDidAppear
- B dealloc

