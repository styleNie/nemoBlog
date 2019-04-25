---
title: "Python xlwt写excel文件"    
author:     
date: June 05, 2018    

toc:     
  depth_from: 1    
  depth_to: 6    
  ordered: false    
  ignoreLink:false    
  
html:    
  embed_local_images: true    
  embed_svg: true    
  offline: false    
  toc: false    

print_background: false    

export_on_save:    
  html: true    

---

# <center>Python xlwt写excel文件</center>


```python
import os
import time   
import xlwt                                                       
from xlwt import *

filename = 'TestData2.xls'                                      #检测当前目录下是否有TestData2.xls文件，如果有则清除以前保存文件
if os.path.exists(filename):
    os.remove(filename)

print(time.strftime("%Y-%m-%d",time.localtime(time.time())))    #打印读取到当前系统时间    

wbk = Workbook(encoding='utf-8')  
sheet = wbk.add_sheet('new sheet 1', cell_overwrite_ok=True)                 #第二参数用于确认同一个cell单元是否可以重设值,需要合并单元格时必须设为True
style = XFStyle()                       #赋值style为XFStyle()，初始化样式                                    

for i in range(0x00,0xff):              # 设置单元格背景颜色                                 
    pattern = Pattern()                 # 创建一个模式                                        
    pattern.pattern = Pattern.SOLID_PATTERN     # 设置其模式为实型              
    pattern.pattern_fore_colour = i                             
    # 设置单元格背景颜色 0 = Black, 1 = White, 2 = Red, 3 = Green, 4 = Blue, 5 = Yellow, 6 = Magenta,  the list goes on...
    style.pattern = pattern             # 将赋值好的模式参数导入Style                                  
    Line_data = (u'测试表')              #创建一个Line_data列表，并将其值赋为测试表，以utf-8编码时中文前加u                                 
    sheet.write_merge(i, i, 0, 2, Line_data, style) #以合并单元格形式写入数据，即将数据写入以第1/2/3列合并德单元格内         

for i in range(0x00,0xff):              # 设置单元格内字体样式                            
    fnt = Font()                                # 创建一个文本格式，包括字体、字号和颜色样式特性                              
    fnt.name = u'微软雅黑'                      # 设置其字体为微软雅黑                                 
    fnt.colour_index = i                        # 设置其字体颜色                                    
    fnt.bold = True                                             
    style.font = fnt                            #将赋值好的模式参数导入Style                                   
    sheet.write_merge(i,i,3,5,Line_data,style)  #以合并单元格形式写入数据，即将数据写入以第4/5/6列合并德单元格内                    

for i in range(0, 0x53):                # 设置单元格下框线样式                                    
    borders = Borders()                                         
    borders.left = i                                           
    borders.right = i                                           
    borders.top = i                                             
    borders.bottom = i                                          
    style.borders = borders         #将赋值好的模式参数导入Style                                 
    sheet.write_merge(i,i,6,8,Line_data,style)  #以合并单元格形式写入数据，即将数据写入以第4/5/6列合并的单元格内

# ----------设置列宽高--------------
"""
xlwt中列宽的值表示方法：默认字体0的1/256为衡量单位。
xlwt创建时使用的默认宽度为2960，既11个字符0的宽度
所以我们在设置列宽时可以用如下方法：
width = 256 * 20    256为衡量单位，20表示20个字符宽度
"""
        
for i in range(6, 80):                  # 设置单元格下列宽样式                                        
    sheet.write(0,i,Line_data,style)
    sheet.col(i).width = 0x0d00 + i*50
    
fmts  = [
            'general',
            '0',
            '0.00',
            '#,##0',
            '#,##0.00',
            '"$"#,##0_);("$"#,##0)',
            '"$"#,##0_);[Red]("$"#,##0)',
            '"$"#,##0.00_);("$"#,##0.00)',
            '"$"#,##0.00_);[Red]("$"#,##0.00)',
            '0%',
            '0.00%',
            '0.00E+00',
            '# ?/?',
            '# ??/??',
            'M/D/YY',
            'D-MMM-YY',
            'D-MMM',
            'MMM-YY',
            'h:mm AM/PM',
            'h:mm:ss AM/PM',
            'h:mm',
            'h:mm:ss',
            'M/D/YY h:mm',
            '_(#,##0_);(#,##0)',
            '_(#,##0_);[Red](#,##0)',
            '_(#,##0.00_);(#,##0.00)',
            '_(#,##0.00_);[Red](#,##0.00)',
            '_("$"* #,##0_);_("$"* (#,##0);_("$"* "-"_);_(@_)',
            '_(* #,##0_);_(* (#,##0);_(* "-"_);_(@_)',
            '_("$"* #,##0.00_);_("$"* (#,##0.00);_("$"* "-"??_);_(@_)',
            '_(* #,##0.00_);_(* (#,##0.00);_(* "-"??_);_(@_)',
            'mm:ss',
            '[h]:mm:ss',
            'mm:ss.0',
            '##0.0E+0',
            '@'
    ]
i =0
for fmt in fmts:
    style.num_format_str = fmt                # 设置字符显示格式
    sheet.write(i,9,-123456789.01234,style)   # 数字太大时，显示乱码，这时需要调整加大列宽
    i +=1

#设置行高度
tall_style = xlwt.easyxf('font:height 720;') # 36pt,类型小初的字号
first_row = sheet.row(0)
first_row.set_style(tall_style)
wbk.save(filename)

# 冻结
# 冻结设置panes_frozen为True，然后设置冻结的位置就好。支持行冻结，列冻结及相关的隐藏功能
from xlwt import *

w =Workbook()
ws1 = w.add_sheet('sheet1')
ws2 = w.add_sheet('sheet2')
ws3 = w.add_sheet('sheet3')
ws4 = w.add_sheet('sheet4')
ws5 = w.add_sheet('sheet5')
ws6 = w.add_sheet('sheet6')

for i in range(0x100):
    ws1.write(i/0x10, i%0x10, i)

for i in range(0x100):
    ws2.write(i/0x10, i%0x10, i)

for i in range(0x100):
    ws3.write(i/0x10, i%0x10, i)

for i in range(0x100):
    ws4.write(i/0x10, i%0x10, i)

for i in range(0x100):
    ws5.write(i/0x10, i%0x10, i)

for i in range(0x100):
    ws6.write(i/0x10, i%0x10, i)

ws1.panes_frozen= True
ws1.horz_split_pos= 2   # 冻结前两行
 
ws2.panes_frozen= True
ws2.vert_split_pos= 2   # 冻结前两列

ws3.panes_frozen= True
ws3.horz_split_pos= 1
ws3.vert_split_pos= 1

ws4.panes_frozen= False
ws4.horz_split_pos= 12
ws4.horz_split_first_visible= 2

ws5.panes_frozen= False
ws5.vert_split_pos= 40
ws4.vert_split_first_visible= 2

ws6.panes_frozen= False
ws6.horz_split_pos= 12
ws4.horz_split_first_visible= 2
ws6.vert_split_pos= 40
ws4.vert_split_first_visible= 2

w.save('panes.xls')
```    



Reference:    
[Python xlwt设置excel单元格字体及格式](https://blog.csdn.net/u013400654/article/details/50284983 )    
[python-excel/xlwt](https://github.com/python-excel/xlwt/tree/master/examples)    
[python之xlwt模块列宽width、行高Heights详解](https://www.cnblogs.com/landhu/p/4978705.html)