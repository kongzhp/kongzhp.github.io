---
layout:     post
title:      "《重构-改善既有代码的设计》读书笔记（2）"
subtitle:   "Long methods "
date:       2015-08-08 17:35:00
author:     "Pete Kong"
header-img: "img/post-bg-02.jpg"
---

# Introduction

Long Methods过长函数是典型的一种代码的“坏味道”，因为函数越长越难理解，而小函数则带来很多便利：

* 解释能力: 良好的函数命名可以很好解释代码的意图，而不需要添加过多的注释说明。
* 共享能力: 一个良好封装的子函数可被多个不同地点调用，从而减少重复代码
* 选择能力：抽出来的小函数可接收多态消息，从而封装条件逻辑，通过把条件逻辑转化成消息形式，即函数参数，降低代码的重复，增加清晰度和提高弹性。



# How To

要把函数变小，大多数情况使用Extract Mehthod即可以。Extract Method，从长函数中提取小粒度的函数十分简单，只要把小函数取一个清晰有意义的名称即可，这里不再赘述。但是抽取的函数可能带来很多的参数和临时变量作为参数，传递给新增的函数，这样可读性几乎没有任何提升。这时可以使用Replace Temp with Query来消除临时元素。还有Introduce Parameter Object消除重复参数组，和Preserve Whole Object来消除过长的参数列。

## Replace Temp with Query

临时变量的缺点在于他们是暂时的，只在函数体里可见，为了使用这个变量，则驱使你写更长的函数。所以如果在一个长函数中，有一个临时变量在很多地方使用，则把临时变量替换为一个查询，那么同一个类中的所有方法都能获得这份信息。

		double getPrice(){
			int basePrice = _quantity * _itemPrice;
			doubel discourntFactor;
			if (basePrice > 1000)
				discourntFactor = 0.95;
			else
				discourntFactor = 0.98;
			return basePrice*discourntFactor;
		}

getPrice()中有两个临时变量，basePrice和discourntFactor，可把他们改为查询函数获得相应的值：
		
		private int basePrice(){
			return _quantity * _itemPrice;
		}
		private double discountFactor(){
			if (basePrice() > 1000)
				return 0.95;
			else 
				reuturn 0.98;
		}

最后getPrice()可简化为

		double getPrice(){
			return basePrice()*discountFactor();
		}

## Introduce Parameter Object

抽取小函数后，有可能发现有一组参数总是被一起传递，这样一组参数被称为Data Clumps(数据泥团)，我们可以使用一个对象包装它们。当把这些参数组织在一起后，有可能发现可以把一些行为移植到新类里，从而使得功能更加内聚，责任更加清晰。

		public class Account {

			private Vector<Entry> _entries = new Vector<Entry>();

			double getFlowBetween(Date start, Date end) {
				double result = 0;
				Enumeration<Entry> e = _entries.elements();
				while (e.hasMoreElements()) {
					Entry each = (Entry) e.nextElement();
					if (each.getDate().equals(start)
							|| each.getDate().equals(end)
							|| (each.getDate().after(start) && each.getDate().before(
									end))) {
						result += each.getValue();
					}
				}
				return result;
			}

			
		}

在Account里不少地方（这里列除了getFlowBetween（））使用了start/end这对参数的方法表示日期范围，这里可以使用Range模式来取代他们：创建DataRange类封装日期范围。另外，循环里头if语句用于判断Entry的日期范围是否包含在参数的日期范围内，这段代码也可以移植到DateRange里，最后代码重构为：

		public class DateRange {

			private final Date _start;
			
			private final Date _end;
			
			DateRange (Date start, Date end) {
				_start = start;
				_end = end;
			}
			
			Date getStart() {
				return _start;
			}
			
			Date getEnd() {
				return _end;
			}
			
			boolean includes (Date arg) {
				return (arg.equals(_start) ||
						arg.equals(_end) ||
						(arg.after(_start) && arg.before(_end)));
			}
		}

		public class Account {

			private Vector<Entry> _entries = new Vector<Entry>();

			double getFlowBetween(DateRange range) {
				double result = 0;
				Enumeration<Entry> e = _entries.elements();
				while (e.hasMoreElements()) {
					Entry each = (Entry) e.nextElement();
					if (range.includes(each.getDate())) {
						result += each.getValue();
					}
				}
				return result;
			}
		}

## Preserve Whole Object

有时候，你会把来自同一个对象的若干个数据项作为参数传递个函数，这样的问题在于，假如此函数的需要该对象的参数发生改变，例如需要这个对象的其他数据项，那么除了修改本函数外，还需要找到调用此函数的所有地方一一修改。而如果直接传递这个对象给这个函数，函数体里再获取此对象的数据项，则可以解决此问题，并且减少参数列，大大增加代码可读性。

		public class Room {
			
			TempRange daysTempRange() {
				return new TempRange();
			}
			
			boolean withinPlan(HeatingPlan plan) {
				int low = daysTempRange().getLow();
				int high = daysTempRange().getHigh();
				return plan.withinRange(low, high);
			}
		}
		public class HeatingPlan {

			private TempRange _range;
			
			boolean withinRange(int low, int high) {
				return (low >= _range.getLow() && high <= _range.getHigh());
			}
		}
		public class TempRange {

			int getHigh() {
				return 100; 
			}
			
			int getLow() {
				return -100;
			}
		}

上述例子中，Room负责记录房间温度，并提供whithinPlan方法告诉客户当前温度是否在计划温度范围内。而HeatingPlan.withinRange方法接收温度范围，这里可以把TempRange对象直接传给它，而不需要拆分。并且计算是否包含温度范围的方法也可以移植到TeamRange当中。

		public class RoomRF {

			TempRangeRF daysTempRange() {
				return new TempRangeRF();
			}
			
			boolean withinPlan(HeatingPlanRF plan) {
				return plan.withinRange(daysTempRange());
			}
		}
		public class HeatingPlanRF {

			private TempRangeRF _range;
			
			boolean withinRange(TempRangeRF roomRange) {
				return (_range.includes(roomRange));
			}
		}
		public class TempRangeRF {

			int getHigh() {
				return 100; 
			}
			
			int getLow() {
				return -100;
			}
			
			boolean includes (TempRangeRF arg) {
				return (arg.getLow() >= this.getLow() && arg.getHigh() <= this.getHigh());
			}
		}
