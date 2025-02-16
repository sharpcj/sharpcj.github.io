---
layout: post
title: "DataSet、DataTable和DataGridView知识备忘"
date:  2016-01-21 16:45:15 +0800
categories: [技术, 编程语言, C#]
tag: [C#, Excel]
---
datatable中，获取第i行j列的单元格内容：
```
string str = DataSet.Tables[0].Rows[i][j].ToString()；
```
datagridview中，获取第i行j列的单元格内容：
```
string str =  DataGridview.Rows[i].Cells[j].Value.ToString()；
```
DataGridview的  SelectionMode 属性 设置为 FullRowSelect 之后，获取指定单元格内容：
```
string str = DataGridview.CurrentRow.Cells[j].Value.ToString() ；
```
DataGridView 单击一次，即进入单元格编辑状态：
```
this.dataGridView1.EditMode = System.Windows.Forms.DataGridViewEditMode.EditOnEnter；
```
datagridview 某一列自动适应列宽：
```
this.dataGridView1.Columns[j].AutoSizeMode = DataGridViewAutoSizeColumnMode.AllCells; 
```

datagridview 中按 enter 键 光标跳至同列下一行。

datatable中添加行
方法一：
```
DataTable  tblDatas = new DataTable("Datas");
DataColumn dc = null;
dc = tblDatas.Columns.Add("ID", Type.GetType("System.Int32"));
dc.AutoIncrement = true;//自动增加
dc.AutoIncrementSeed = 1;//起始为1
dc.AutoIncrementStep = 1;//步长为1
dc.AllowDBNull = false;//

dc = tblDatas.Columns.Add("Product", Type.GetType("System.String"));
dc = tblDatas.Columns.Add("Version", Type.GetType("System.String"));
dc = tblDatas.Columns.Add("Description", Type.GetType("System.String"));

DataRow newRow;
newRow = tblDatas.NewRow();
newRow["Product"] = "水果刀";
newRow["Version"] = "2.0";
newRow["Description"] = "打架专用";
tblDatas.Rows.Add(newRow);

newRow = tblDatas.NewRow();
newRow["Product"] = "折叠凳";
newRow["Version"] = "3.0";
newRow["Description"] = "行走江湖七武器之一";
tblDatas.Rows.Add(newRow);
```

方法二：
```
DataTable tblDatas = new DataTable("Datas");
tblDatas.Columns.Add("ID", Type.GetType("System.Int32"));
tblDatas.Columns[0].AutoIncrement = true;
tblDatas.Columns[0].AutoIncrementSeed = 1;
tblDatas.Columns[0].AutoIncrementStep = 1;

tblDatas.Columns.Add("Product", Type.GetType("System.String"));
tblDatas.Columns.Add("Version", Type.GetType("System.String"));
tblDatas.Columns.Add("Description", Type.GetType("System.String"));

tblDatas.Rows.Add(new object[]{null,"a","b","c"});
tblDatas.Rows.Add(new object[] { null, "a", "b", "c" });
tblDatas.Rows.Add(new object[] { null, "a", "b", "c" });
tblDatas.Rows.Add(new object[] { null, "a", "b", "c" });
tblDatas.Rows.Add(new object[] { null, "a", "b", "c" });
```

将 List<T> 转为 datatable：
```
public static DataTable ToDataTable<T>(IList<T> data)
{
    PropertyDescriptorCollection properties = TypeDescriptor.GetProperties(typeof(T));
    DataTable dt = new DataTable();
    for (int i = 0; i < properties.Count; i++)
    {
        PropertyDescriptor property = properties[i];
        dt.Columns.Add(property.Name, property.PropertyType);
    }
    object[] values = new object[properties.Count];
    foreach (T item in data)
    {
        for (int i = 0; i < values.Length; i++)
        {
            values[i] = properties[i].GetValue(item);
        }
        dt.Rows.Add(values);
    }
    return dt;
}
```