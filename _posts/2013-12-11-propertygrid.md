---
layout: post
title: Property Grid
category: .net
excerpt: .net项目中，针对property grid下拉框的定制
tags: [.net, programming]
---
{% include JB/setup %}

以前在sap实习做项目的时候，开发studio。通过一段时间的学习，掌握了不少实用的技术。比如说，这个property grid的定制。本身并不是很复杂，微软本身提供的editor具备很强的扩展性，只需要你用心地了解其使用方法，大概就齐了。└(^o^)┘  
不用太多废话，贴段代码：  

	 class MyUIEditor:UITypeEditor 
	｛
		public override UITypeEditorEditStyle GetEditStyle(ITypeDescriptorContext context)
		{
			return System.Drawing.Design.UITypeEditorEditStyle.DropDown；
		}//这里有3种editstyle
		
		public override object EditValue(ITypeDescriptorContext context，IServiceProvider provider,object value)
		{
			System.Windows.Forms.Design.IWindowsFormsEditorService iws=
				rSystem.Windows.Forms.Design.IWindowsFormsEditorService)provider.GetService(typeof(System.Windows.Forms.Design.IWindowsFormsEditorService))；
			if(iws!=null)
			{
				mycontrol con = new mycontrol();//因为是自己定制的control理论上可以实现各种要求
				con.setcontainer(iws);//con需要实现这个方法。
				iws.DropDownControl(con);//这里需要绑定
			}
			return value；
		} 
	｝ 
UITypeEditor提供了设计值编辑器的基类，参考<a href="http://msdn.microsoft.com/zh-cn/library/System.Drawing.Design.UITypeEditor(v=vs.110).aspx">MSDN</a>。OK，就如此简单。