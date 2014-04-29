---
layout: post
title: "Never worry about browser caching again"
date: 2014-04-28 12:54:26 -0400
comments: true
categories: [MVC, .NET]
---

<blockquote>
Client: "It's still not working for me." <br />
You: "Hmm.  Have you tried clearing your cache?"<br />
Client: "How do I do that?"<br />
You: "What browser are you using?"<br />
Client: "Blah blah blah"<br />
You: "Blah blah blah"<br />
Client: "Oh NOW it's working!"
</blockquote>
<p>
Had that conversation before? Like one hundred, or one thousand times?  Enough of that.  Trick that there ol' browser to download new JavaScript and Stylesheets after every new deployment and have it cache them until the next one.  All you have to do is change the file names every time.  Nope, that's not something manual you'll have to remember to do.  If you're a .NET developer, make use of MSBuld to do that.  While you're at it, have it minify and combine your resources so that you have one optimized JavaScipt file and one optimized Stylesheet.  All you need is a few additional build tasks that don't come out of the box with Visual Studio.  Grab <a href="http://ajaxmin.codeplex.com/">Mincrosoft Ajax Minifier</a> and the <a href="https://msbuildextensionpack.codeplex.com/" target="_blank">MSBuild Extension Pack</a>.  If your not a .NET developer then I bet there's a similar way to do this in your own language.
</p>

<p>
	Once you've got that installed, unload your project in Visual Studio so that you can edit the project file.  Down at the bottom of it, just before the closing &lt;/Project&gt; tag, add the following target:
</p>

{% codeblock lang:xml %} 
<Target Name="BeforeBuild">
<ItemGroup>
  <GeneratedCSSJS Include="Content/combined.*.css" />
  <GeneratedCSSJS Include="Scripts/combined.*.js" />
</ItemGroup>
<Delete Files="@(GeneratedCSSJS)" />
</Target>
{% endcodeblock %}

<p>
This is going to do cleanup from previous builds.  Otherwise we'd end up with two new files after each build.  
</p>

<p>Next we are going to add another Target named AfterBuild.  This is going to contain the meat and potatoes. The following code snippets will all go inside this element.</p>

{% codeblock lang:xml %} 
<Target Name="AfterBuild">
		This is where the rest of our work is going to go.
</Target>
{% endcodeblock %}

 <p>
We're going to be using the assembly version number in our file names so that they change in a predictable way on each build.  This is how the style and script references in our HTML will know what file name to refer to.  It's possible that a deployment might not include any code changes, and to ensure we get a new assembly version number we need to force a compile on each build.  We do that by creating a dummy .cs file in our project and <em>touch</em> it to force a re-compile.  
 </p>


{% codeblock lang:xml %} 
<Exec Command="ATTRIB -R Build.cs" />
<Touch Files="Build.cs" />
<Exec Command="ATTRIB +R Build.cs" />
{% endcodeblock %}

<p>Next we'll use a task from the extension pack to get at the assembly info.</p>

{% codeblock lang:xml %} 
<MSBuild.ExtensionPack.Framework.Assembly TaskAction="GetInfo" NetAssembly="$(OutputPath)\MyAssembly.dll">
  <Output TaskParameter="OutputItems" ItemName="Info" />
</MSBuild.ExtensionPack.Framework.Assembly>
{% endcodeblock %}

<p>
Now we'll declare which files we want to minify and concatenate. 
</p>
{% codeblock lang:xml %} 
<ItemGroup>
  <CSSMin Include="Content/main.css" />
  <CSSMin Include="Content/additional.css" />
</ItemGroup>
<ItemGroup>
  <CSSCat Include="Content/main.css" />
  <CSSCat Include="Content/additional.css" />
</ItemGroup>
<ItemGroup>
  <JSMin Include="Scripts\global.js" />
  <JSMin Include="Scripts\additional.js" />
</ItemGroup>
<ItemGroup>
  <JSCat Include="Scripts\global.js" />
  <JSCat Include="Scripts\additional.js" />
</ItemGroup>


 {% endcodeblock %}

 <p>
Now minify, combine, and write to a single css and js file, including the assembly version in the file name.
 </p>

 {% codeblock lang:xml %} 
<AjaxMin JsSourceFiles="@(JSMin)" JsSourceExtensionPattern="\.js$" JsTargetExtension=".min.js" CssSourceFiles="@(CSSMin)" CssSourceExtensionPattern="\.css$" CssTargetExtension=".min.css" />
    <ReadLinesFromFile File="%(JSCat.Identity)">
  <Output TaskParameter="Lines" ItemName="JSLines" />
</ReadLinesFromFile>
<WriteLinesToFile File="Scripts/combined.%(Info.AssemblyVersion).min.js" Lines="@(JSLines)" Overwrite="true" />
<ReadLinesFromFile File="%(CSSCat.Identity)">
  <Output TaskParameter="Lines" ItemName="CSSLines" />
</ReadLinesFromFile>
<WriteLinesToFile File="Content/combined.%(Info.AssemblyVersion).min.css" Lines="@(CSSLines)" Overwrite="true" />

 {% endcodeblock %}

 <p>
Lastly, let's declare the resulting files as part of the project in case we are using Visual Studio's publishing feature.
 </p>

  {% codeblock lang:xml %} 
<ItemGroup>
  <Content Include="Scripts/combined.%(Info.AssemblyVersion).min.js" />
  <Content Include="Content/combined.%(Info.AssemblyVersion).min.css" />
</ItemGroup>
   {% endcodeblock %}

<p>
There's one more part of this.  We'll need to dynamically reference these files from our views.  The more simple way to do this would be to go into our layout file and add some references like the following:
</p>

 {% codeblock lang:html %} 
 {
   var version = Assembly.GetExecutingAssembly().GetName().Version.ToString();
 }

<link href=/Content/combined.@(version).min.css rel="stylesheet" />
<script type="text/JavaScipt" src="/Scripts/combined.@(version).min.js"></script>
{% endcodeblock %}

<p>
That works but my front end guys aren't going to like trying to debug the minified and combined files.  I like to add an additional step so that this optimization isn't used in our development environments.  We can do this by referring to the original unminified files when the DEBUG constant is defined.  I generally like to create a class for this so as to keep my layout clean.  Just in case I need to debug in production I make this work when a <em>debug</em> query parameter is present as well.  The following C# class is an example of how I do this:
</p>

 {% codeblock lang:csharp %} 

    public static class WebResources
    {
        private static readonly string ExpandedScripts;
        private static readonly string CompressedScripts;
        private static readonly string ExpandedCssFiles;
        private static readonly string CompressedCssFiles;


        static WebResources()
        {

            var expandedScriptsBuilder = new StringBuilder();
            expandedScriptsBuilder.AppendLine(ScriptTag("/Scripts/global.js"));
            expandedScriptsBuilder.AppendLine(ScriptTag("/Scripts/additionaljs"));
            ExpandedScripts = expandedScriptsBuilder.ToString();

            ExpandedCssFiles = CssLink("/Content/main.css") + CssLink("/Content/additional.css")

            var version = Assembly.GetExecutingAssembly().GetName().Version.ToString();
            CompressedCssFiles = CssLink(string.Format("/Content/combined.{0}.min.css", version));
            CompressedScripts = ScriptTag(string.Format("/Scripts/combined.{0}.min.js", version));
            


        }

        public static string JavaScripts
        {
            get
            {

                if (IsDebugMode)
                {
                    return ExpandedScripts;
                }

#if DEBUG
                return ExpandedScripts;
#else
                return CompressedScripts;
#endif
            }
        }



        public static string CssFiles
        {
            get
            {
#if DEBUG
                return ExpandedCssFiles;
#else
                return IsDebugMode ? ExpandedCssFiles : CompressedCssFiles;
#endif
            }
        }

        private static string ScriptTag(string url)
        {
            return string.Format("<script src=\"{0}\"></script>", url);
        }

        private static string CssLink(string url)
        {
            return string.Format("<link href=\"{0}\" rel=\"stylesheet\"/>", url);
        }

        private static bool IsDebugMode
        {
            get
            {
                return HttpContext.Current.Request.QueryString["debug"] == "true";
            }
        }
    }

 {% endcodeblock %}

<p>
Then of course I reference these static properties in the layout like the following.
</p>

 {% codeblock lang:html %} 

<link href=/Content/combined.@(WebResources.CssFiles).min.css rel="stylesheet" />
<script type="text/JavaScipt" src="/Scripts/combined.@(WebResources.Javascripts).min.js"></script>
{% endcodeblock %}

<p>
What's beautiful about this is that we can configure the web server to tell the browsers to cache until their hearts are content.  They'll download the resource files exactly once until the next deployment occurs.
</p>
<p>
Oh, one other thing I wanted to mention.  Some of you might say <em>I handle this problem myself by just adding a dynamic query parameter to the script or style reference</em>. This is certainly a shortcut that I've seen in practice but it doesn't always work because some browsers don't respect query parameters in Stylesheet and JavaScript references.  In some instances I've seen random numbers appended as query parameters on each page request, but if that succeeds in forcing a download, it's inefficient because the browser will download the file over and over on each page view.
</p>