---
title: 字符在字符串中出现的频率统计
date: 2019-07-04 09:39:13
categories:
- AC
---
问题描述：
> 统计字符串中每一个字符在该字符串中出现的次数，按次数从高至低排序输出，若次数相同，则按在该字符串中出现的顺序输出，区分大小写。

解题思路：
> 此题分为两部分，统计字符出现次数，可以用LinkedHashMap来统计；按次数从高至低输出，则需要对map的value进行排序，可以利用Comparator比较器。

<!-- more -->
具体实现代码如下：
{% codeblock testAC1.java lang:java %}
import java.util.ArrayList;
import java.util.Collections;
import java.util.Comparator;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.Map.Entry;
import java.util.Scanner;

public class testAC1 {
	public static void main(String[] Args) {
		Scanner sc = new Scanner(System.in);
		while (sc.hasNextLine()) {
			String str = sc.nextLine();
			Map<Character, Integer> map = new LinkedHashMap<Character, Integer>();
			// 利用map统计字符出现次数
			for (int i = 0; i < str.length(); i++) {
				char ch = str.charAt(i);
				if (map.containsKey(ch)) {
					map.put(ch, map.get(ch) + 1);
				} else {
					map.put(ch, 1);
				}
			}
			// 用Comparator比较器排序
			List<Map.Entry<Character, Integer>> list = new ArrayList<Map.Entry<Character, Integer>>(map.entrySet());
			Collections.sort(list, new Comparator<Map.Entry<Character, Integer>>() {
				@Override
				public int compare(Entry<Character, Integer> o1, Entry<Character, Integer> o2) {
					return o2.getValue().compareTo(o1.getValue());
				}
			});
			for (Map.Entry<Character, Integer> m : list) {
				System.out.println(m.getKey() + "=" + m.getValue());
			}
		}
		sc.close();
	}
}
{% endcodeblock %}
