---
layout:     post
title:      "《重构-改善既有代码的设计》读书笔记（1）"
subtitle:   "Switch Statements"
date:       2015-08-01 17:35:00
author:     "Pete Kong"
header-img: "img/post-bg-02.jpg"
---

# Introduction

《重构》这本书买了好久了，一开始看了一些，当时写代码写得少，看了也忘了，就没什么心思看。最近几乎每天都有review代码的机会，重新拾起这本书，写下读书笔记，希望能给大家提供一些帮助。

作者（Martin Fowler）把需要重构的代码成为有“坏味道”的代码，而下文所述的Switch Statements就是其中一种“坏味道”。

# Switch Statements

Maritin认为switch语句的问题在于重复，你会发现同样的switch语句散布在不同的地方，如果你要为它添加一个case语句，你就需要找到所有的switch并修改它们。除此之外，我认为如果条件越多，那么函数就会变得很长，同样会降低程序的可阅读性。而面向对象的多态概念可为此带来优雅的解决办法。

替换Switch语句一般分为两步走：1.Replace Type Code with State/Stategy 2. Replace Condition with Polymorphism. 下面将以一计算雇员工资的例子说明这两步方法。

		public class Employee {

			private int _type;
			static final int ENGINEER = 0;
			static final int SALESMAN = 1;
			static final int MANAGER = 2;

			Employee(int type) {
				_type = type;
			}

			int _monthlySalary;
			int _commission;
			int _bonus;
			
			int payAmount() {
				switch (_type) {
				case ENGINEER:
					return _monthlySalary;
				case SALESMAN:
					return _monthlySalary + _commission;
				case MANAGER:
					return _monthlySalary + _bonus;
				default:
					throw new RuntimeException("Incorrect Employee");
				}
			}
			
			
		}

1. Replace Type Code with State/Stategy

	第一步把状态码(雇员类型)从宿主类（Employee）中抽出来，创建三个子类Engineer、Salesman、Manager，并提供一工厂方法创建子类。这里的工厂方法还是存在switch，但是只存在一个地方，以后增加Employee类型也只需要新增一个EmployeeType的子类，并在newType方法中新增创建该子类对象的case。

		abstract class EmployeeType {
			
			static final int ENGINEER = 0;
			static final int SALESMAN = 1;
			static final int MANAGER = 2;
			
			abstract int getTypeCode();
			
			static EmployeeType newType(int code) throws IllegalAccessException {
				switch(code) {
				case ENGINEER:
					return new Engineer();
				case SALESMAN:
					return new Salesman();
				case MANAGER:
					return new Manager();
				default:
					throw new IllegalAccessException("Incorrect Employee Code");
				}
			}
		}


		class Engineer extends EmployeeType{
			
			int getTypeCode() {
				return ENGINEER;
			}
			
		}

		public class Salesman extends EmployeeType{
	
			int getTypeCode() {
				return SALESMAN;
			}
		}



2.  Replace Condition with Polymorphism

	把类型码抽出来后，计算工资的职责就可以转移到具体的EmployeeType子类中了，而Employee类中的payAmount()中的一坨switch语句则变为对EmployeeType的payAmount方法调用。

		public class Salesman extends EmployeeType{
		
			int payAmount(EmployeeRF emp) {
				return emp.get_monthlySalary() + emp.get_commission();
			}
		}

		public class EmployeeRF {

			private EmployeeType _type;

			EmployeeRF(int type) {
				try {
					_type = EmployeeType.newType(type);
				} catch (IllegalAccessException e) {
					e.printStackTrace();
				}
			}
			
			int _monthlySalary;
			int _commission;
			int _bonus;

			int payAmount() {
				return _type.payAmount(this);
			}

			public int get_monthlySalary() {
				return _monthlySalary;
			}

			public void set_monthlySalary(int monthlySalary) {
				_monthlySalary = monthlySalary;
			}

			public int get_commission() {
				return _commission;
			}

			public void set_commission(int commission) {
				_commission = commission;
			}

			public int get_bonus() {
				return _bonus;
			}

			public void set_bonus(int bonus) {
				_bonus = bonus;
			}

		}		

# Step Forward

文中使用整形常量来表示类型码，然而根据《Effective Java》所述，使用整形常量比较脆弱，如果与枚举常量关联的int发生了变化，客户端需重新编译。另外没有便利的方法将常量名称打印出来。

因此，我认为可以稍微修改成这样：

		public enum EmployeeTypeEnum {
			ENGINEER, MANAGER, SALESMAN;
		}

		abstract class EmployeeType {

			abstract int payAmount(EmployeeRF emp)
			
			static EmployeeType newType(EmployeeTypeEnum code) throws IllegalAccessException {
				switch(code) {
				case ENGINEER:
					return new Engineer();
				case SALESMAN:
					return new Salesman();
				case MANAGER:
					return new Manager();
				default:
					throw new IllegalAccessException("Incorrect Employee Code");
				}
			}
		}

		public class Salesman extends EmployeeType{
		
			public int payAmount(EmployeeRF emp) {
				return emp.get_monthlySalary() + emp.get_commission();
			}
		}

		public class EmployeeRF {

			private EmployeeTypeEnum _type;

			EmployeeRF(EmployeeTypeEnum type) {
				try {
					_type = EmployeeType.newType(type);
				} catch (IllegalAccessException e) {
					e.printStackTrace();
				}
			}
			
			int _monthlySalary;
			int _commission;
			int _bonus;

			int payAmount() {
				return _type.payAmount(this);
			}

			public int get_monthlySalary() {
				return _monthlySalary;
			}

			public void set_monthlySalary(int monthlySalary) {
				_monthlySalary = monthlySalary;
			}

			public int get_commission() {
				return _commission;
			}

			public void set_commission(int commission) {
				_commission = commission;
			}

			public int get_bonus() {
				return _bonus;
			}

			public void set_bonus(int bonus) {
				_bonus = bonus;
			}

		}

