﻿Title: Automatically Generating Column Titles For A KendoUI MVC Grid
Published: 4/11/2013
Tags:
  - ASP.NET
  - ASP.NET MVC
  - KendoUI
  - KendoUI MVC
  - grid
  - data annotations
---
<p>I love KendoUI, especially because of the available MVC wrappers. It is a very well engineered product with lots of opportunity for extension, and in this post I'll briefly discuss one that should relieve a small pain point: generating grid column titles from a <code><a href="http://msdn.microsoft.com/en-us/library/system.componentmodel.dataannotations.displayattribute.aspx">DisplayAttribute</a></code> <a href="http://msdn.microsoft.com/en-us/library/dd901590(v=vs.95).aspx">data annotation</a>. As you probably already know, you can change the way your UI layer presents properties of your model by applying the <code>DisplayAttribute</code> data annotation. This causes most of the UI code to use the <code>Name</code> property of the attribute when displaying that property. It looks like this:</p>

<pre class="prettyprint">[Display(Name = "The Product!")]
public string ProductName { get; set; }</pre>

<p>Until recently, KendoUI would not recognize the <code>DisplayAttribute</code> applied to a bound column. However, before I go much further, it's worth noting that this is no longer the case. I waited too long to post this article and KendoUI already gets the column title from a <code>DisplayAttribute</code> data annotation if there is one. I am posting this anyway because the technique could be generalized to other ways of customizing the grid by using an extension method.</p>

<p>Now, let's say we have a KendoUI grid declared in Razor using the MVC wrappers (this is from their demo page):</p>

<pre class="prettyprint">@model IEnumerable&lt;Kendo.Mvc.Examples.Models.ProductViewModel&gt;

@(Html.Kendo().Grid(Model)    
    .Name("Grid")
    .Columns(columns =&gt;
    {
        columns.Bound(p =&gt; p.ProductID);
        columns.Bound(p =&gt; p.ProductName);
    })
    .DataSource(dataSource =&gt; dataSource
        .Ajax().Read(read =&gt; read.Action("Products_Read", "Grid"))
    )
)</pre>

<p>When displaying the column titles, KendoUI will use the name of the property to generate the name of the column. In the example above, you will get two columns named "Product ID" and "Product Name" (Kendo is smart enough to add the spaces in between the capital letters). But what if we wanted our second column to be named "The Product!" as in the example application of <code>DisplayAttribute</code> above? We could add the title explicitly using the <code>Title()</code> extension method:</p>

<pre class="prettyprint">...
columns.Bound(p =&gt; p.ProductName).Title("The Product!");
...</pre>

<p>But this violates the <a href="http://en.wikipedia.org/wiki/Don't_repeat_yourself">DRY principle</a>. If you want to change the title in the future, you'll need to remember to change it in both places. What would be better is if we could write something like:</p>

<pre class="prettyprint">...
columns.Bound(p =&gt; p.ProductName).DisplayNameTitle();
...</pre>

<p>This would indicate to the grid that the title should be set by getting a <code>DisplayAttribute</code> <code>Name</code> property (if there is one) and using the normal name generation otherwise. The code for such an extension method is below:</p>

<pre class="prettyprint">public static GridBoundColumnBuilder&lt;TModel&gt; DisplayNameTitle&lt;TModel&gt;(
    this GridBoundColumnBuilder&lt;TModel&gt; builder) where TModel : class, new()
{
    // Create an adapter to access the typed grid column
    // (which contains the Expression)
    Type adapterType = typeof(GridBoundColumnAdapter&lt;,&gt;)
        .MakeGenericType(typeof(TModel), builder.Column.MemberType);
    IGridBoundColumnAdapter adapter =
        (IGridBoundColumnAdapter)Activator.CreateInstance(adapterType);

    // Use the adapter to get the title and set it
    return builder.Title(adapter.GetDisplayName(builder.Column));
}

private interface IGridBoundColumnAdapter
{
    string GetDisplayName(IGridBoundColumn column);
}

private class GridBoundColumnAdapter&lt;TModel, TValue&gt;
    : IGridBoundColumnAdapter where TModel : class, new()
{
    public string GetDisplayName(IGridBoundColumn column)
    {
        // Get the typed bound column
        GridBoundColumn&lt;TModel, TValue&gt; boundColumn =
            column as GridBoundColumn&lt;TModel, TValue&gt;;
        if (boundColumn == null) return String.Empty;

        // Create the appropriate HtmlHelper and use it to get the display name
        HtmlHelper&lt;TModel&gt; helper = HtmlHelpers.For&lt;TModel&gt;(
            boundColumn.Grid.ViewContext,
            boundColumn.Grid.ViewData,
            new RouteCollection());
        return helper.DisplayNameFor(boundColumn.Expression).ToString();
    }
}</pre>

<p>So let's look at this code a little more closely. The first <code>DisplayNameTitle&lt;TModel&gt;()</code> method is the actual extension. It takes a <code>GridBoundColumnBuilder&lt;TModel&gt;</code> because that's what the <code>Bound()</code> method returns as part of the KendoUI MVC fluent interface for column specifications. The <code>DisplayNameTitle</code> extension method creates an instance of an adapter class that can be used to manipulate the grid column. Because the KendoUI MVC classes are strongly typed, we need to use reflection to create an adapter with the proper generic type parameters. The key to this working is that the adapter class implements the non-generic <code>IGridBoundColumnAdapter</code> interface, which means that by casting the reflection-generated generic adapter class to the interface we can call non-generic methods that have access to the generic type parameters we used during the reflected construction in the actual implementation of the method.</p>

<p>The real work gets done inside the <code>GetDisplayName()</code> implementation. This method <a href="/posts/getting-an-htmlhelper-for-an-alternate-model-type">creates an appropriately typed HtmlHelper</a> and then uses it to call the <code>HtmlHelper.DisplayNameFor()</code> extension method. This ensures that our own <code>DisplayNameTitle()</code> extension will always return the exact same title that would be returned if we used the normal MVC <code>HtmlHelper</code> methods to display the property.</p>

<p>This technique could be used to add other extensions to the KendoUI column building fluent interface as well. For example, you could automatically make a column sortable or not based on the data type.</p>
