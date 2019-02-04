Title:Currency

---

# 一. Currency简介

Currency是一个处理各个国家货币的类，可以根据Locale来选择不同的地区货币，比如中国:CHINA，美国:US

当然，我们还可以选择查看支持的所有地区货币列表

由于它的构造函数已经私有化，所以只能通过它公开的getInstance来访问。



# 二. 使用方法

```
Currency cur = Currency.getInstance(Locale.JAPAN);
    	
System.out.println(cur.getDisplayName());
System.out.println(cur.getCurrencyCode());
System.out.println(cur.getDefaultFractionDigits());
System.out.println(cur.getSymbol());

//显示所支持的所有的地区货币列表
for (Currency item : cur.getAvailableCurrencies()) {
	System.out.println(item.getDisplayName());
}
```

输出

```
日元
JPY
0
JP¥

巴哈马元
津巴布韦元 (2008)
不丹努尔特鲁姆
突尼斯第纳尔
亚美尼亚德拉姆
卢森堡法郎
Sucre
特别提款权
西非法郎
哥伦比亚币
意大利里拉
叙利亚镑
阿联酋迪拉姆
尼泊尔卢比
布隆迪法郎
斯里兰卡卢比
加拿大元
瑞典克朗
白俄罗斯新卢布 (1994–1999)
萨尔瓦多科朗
欧洲计算单位 (XBC)
白俄罗斯卢布
尼加拉瓜科多巴
马其顿第纳尔
汤加潘加
几内亚比绍比索
赞比亚克瓦查
乌干达先令
圭亚那元
白俄罗斯卢布 (2000–2016)
法郎 (WIR)
旧罗马尼亚列伊
乌拉圭比索
委内瑞拉玻利瓦尔 (1871–2008)
摩尔多瓦列伊

后面就继续贴了...
```







