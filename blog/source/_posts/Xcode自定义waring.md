---
title: Xcode自定义waring
categories: iOS
tags: 奇巧淫技
date: 2017-05-26 18:55:01

---

![添加方法](http://7xvz3k.com1.z0.glb.clouddn.com/xcode%E8%87%AA%E5%AE%9A%E4%B9%89%E8%84%9A%E6%9C%AC.png)

```
KEYWORDS="\/\/TODO|\/\/FIXME|\/\/\?\?\?:|\/\/\!\!\!:"
	find "${SRCROOT}" \( -name "*.h" -or -name "*.m" -or -name "*.swift" \) -print0 | \
	xargs -0 egrep --with-filename --line-number --only-matching "($KEYWORDS).*\$" | \
	perl -p -e "s/($KEYWORDS)/ warning: \$1/"
```
