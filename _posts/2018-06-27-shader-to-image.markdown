---
layout: post
title:  "Shader渲染到图片"
date:   2018-06-27 17:30:06 +0800
author: codinggamer
categories: Unity
---
最近在想制作Gameboy风格的平台游戏，所以遇到一个需求就是需要将本来不是Gameboy风格的图片转换成Gameboy风格。因为之前购买了fx2d的插件，里面具有Gameboy的shader，只需要赋值到SpriteRenderer上即可。所以主要问题就是在Unity中将其图片导出到本地。

### 这样制作的好处

运行效率比较高，通过牺牲内存来换取GPU性能，如果比较多的精灵使用一些耗费性能的静态滤镜Shader可以通过这样优化使得不至于发热变卡。

批量处理，比如你需要把图片置灰，如果让美工一个一个处理还是比较低效。

### 最后输出的结果
比如原图是这个样子

![](/assets/tmp114036d2.png)

最后导出的是这样

![](/assets/tmp01d94655.png)

### 涉及到的知识
* RenderTexture通过Graphics.Blit截取图片
* unity编辑器拓展wizard弹窗进行批量创建


### 处理方法简介
通过Graphics.Blit进行把图片纹理绘制到RenderTexture，然后将RendterTexture纹理绘制到Texture2D中然后保存到本地。

很简单，下面结合代码讲解下。

### 具体代码
下面这个核心的逻辑，通过下面代码可以将图片和所需要的材质合并然后输出转换到的图片
{% highlight C# %}

			//初始需要转换的图片
			Texture2D tex = originTexture;
			
			//生成一个转换完成之后的图片
			Texture2D out_tex = new Texture2D( tex.width, tex.height );
			
			//生成一个RendterTexture
			RenderTexture rt = new RenderTexture( tex.width, tex.height, 24, RenderTextureFormat.ARGB32, RenderTextureReadWrite.Linear );
			//将初始的图片绘制到RenderTexture上面，最后一个参数为转换用到的材质
			Graphics.Blit( tex ,  rt , material );
			
			//然后将RendterTexture绘制到图片上去
			RenderTexture.active = rt;
			out_tex.ReadPixels( new Rect( 0,0, tex.width, tex.height ), 0,0 , false );
			out_tex.Apply();
			RenderTexture.active = null;
			
			//保存到本地
			string path = Application.dataPath + "/RenderTexture/Texture/output/" + tex.name + ".png";
			System.IO.File.WriteAllBytes( path, out_tex.EncodeToPNG() );

{% endhighlight %}

下面说说使用Wizard弹窗进行批量处理，在脚本文件夹创建Editor文件，然后创建一个脚本

{% highlight C# %}
	using System.Collections;
	using System.Collections.Generic;
	using UnityEngine;
	using UnityEditor;
	using System.IO;
	public class SpriteRenderToTexture2DWizard : ScriptableWizard
	{
		//初始的图片，使用一个集合保存
		public Texture2D[] originTextures;

		public Material need_use_material;

		public string out_folder = "RenderTexture/Texture/output";

		public Object folder_obj;

		[MenuItem( "PureUtils/SpriteRenderToTexture2D" )]
		protected static void SpriteRenderToTexture2D()
		{
			ScriptableWizard.DisplayWizard<SpriteRenderToTexture2DWizard>( "将SpriteRender转化成对应的Texture2D", "生成" );
		}

		/// <summary>
		/// OnGUI is called for rendering and handling GUI events.
		/// This function can be called multiple times per frame (one call per event).
		/// </summary>
		void OnGUI()
		{
			ScriptableObject target = this;
			SerializedObject so = new SerializedObject( target );
			SerializedProperty property = so.FindProperty( "originTextures" );

			EditorGUIUtility.LookLikeInspector();
			EditorGUI.BeginChangeCheck();
			EditorGUILayout.PropertyField( property, true );
			if(EditorGUI.EndChangeCheck())
            	so.ApplyModifiedProperties();
			EditorGUIUtility.LookLikeControls();

			need_use_material = (Material)EditorGUILayout.ObjectField( "需要附加的材质", need_use_material, typeof( Material ), true, GUILayout.Height( 20 ) );
			folder_obj = EditorGUILayout.ObjectField( "文件夹位置", folder_obj, typeof( Object ), false, GUILayout.Height( 20 )  );
			
			string path = AssetDatabase.GetAssetPath( folder_obj );
			EditorGUILayout.LabelField( "path: " +  path );
			if( Directory.Exists( path ) )
			{
				EditorGUILayout.LabelField( "文件夹位置: " +  out_folder );
			}
			else
			{
				EditorGUILayout.LabelField( "必须是文件夹位置" );
			}

			if( GUILayout.Button( "创建", GUILayout.Width( 200 ) ) )
			{
				OnWizardCreate();
			}

		}

		void OnWizardCreate () {
			Debug.Log("OnWizardCreate");
			out_folder = Application.dataPath + "/" + out_folder;

			foreach (var originTexture in originTextures)
			{
				
				string path =   out_folder + "/" + originTexture.name + ".png";

				Debug.Log("path:" + path);
				if( !Directory.Exists( out_folder ) )
				{
					Directory.CreateDirectory( out_folder );
				}

				Texture2D out_tex = new Texture2D( originTexture.width, originTexture.height );

				RenderTexture rt = new RenderTexture( originTexture.width, originTexture.height, 24, RenderTextureFormat.ARGB32, RenderTextureReadWrite.Linear );

				Material _material = need_use_material;
				Graphics.Blit( originTexture, rt, _material );

				RenderTexture.active = rt;
				out_tex.ReadPixels( new Rect( 0,0, originTexture.width, originTexture.height ), 0,0, false );
				out_tex.Apply();
				RenderTexture.active = null;

				// if( !File.Exists( path ) )
				// {
					File.WriteAllBytes( path, out_tex.EncodeToPNG() );
				// }
			}

			AssetDatabase.Refresh();
		}
		void OnWizardUpdate () {

		}
		// When the user pressed the "Apply" button OnWizardOtherButton is called.
		//当用户按下"Apply"按钮，OnWizardOtherButton被调用
		void OnWizardOtherButton () {
			
		}

		
	}
	
{% endhighlight %}
因为我需要拖拽文件夹，所以重写了OnGUI方法。实际运行如下所示

![](/assets/shaderTextureToTexture.gif)