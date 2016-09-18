---
layout: post
title: Unityで使える非同期処理アセットとかクラスとか
published: true
---

村岡です。CoRoutineやThreadなどで非同期的なことをしたいとき、ケースに応じて使えるアセットとかライブラリをまとめておこうと思います。  
とりあえず今まで試したものを忘れないように。

# Thead Ninja

[https://www.assetstore.unity3d.com/jp/#!/content/15717](https://www.assetstore.unity3d.com/jp/#!/content/15717)

CoRoutineな書き方で非同期処理を完結に書ける。かなり使い勝手がいい。個人的にけっこう好み。
どうやらIEmumeratorの中身はThreadで実行される仕組みになっている模様です。
Task.Cancel() などでIEmumerator外部から実行中のタスクを終了するなどできる。

## Code Example

```csharp
using UnityEngine;
using UnityEngine.UI;
using System.Collections;
using System.Threading;
using CielaSpike;

public class CorutineTest : MonoBehaviour {

	private bool mGo = true;
        private int mCount = 0;
	private Task task;

	// Use this for initialization
	void Start () {
		Debug.Log("Start");
    		this.StartCoroutineAsync(Sample(), out task);    // task に Sample() の実行タスクが返る
		Debug.Log("Running...");
	}

	private IEnumerator Sample() {  
		while (mGo) {
			Thread.Sleep(2000);
			mCount++;
			Debug.Log("Count: " + mCount.ToString());
		}
		yield return null;
	}
	
	// Update is called once per frame
	void Update () {
		if(mCount > 8 && mGo == true) {    // カウント9 でSampleタスクをキャンセルする
			task.Cancel();
			mGo = false;
			Debug.Log("Task canceled");
			Debug.Log(task.State);   // ステート Canceled が表示される
		}
	}

	// アプリケーション終了イベント
	void OnApplicationQuit() {
		mGo = false;   // mGo を falseにしないとアプリ終了後もタスクが実行中になる
		Debug.Log("Application is terminated.");
	}
}
```

# Spicy Pixel

[https://www.assetstore.unity3d.com/jp/#!/content/3586](https://www.assetstore.unity3d.com/jp/#!/content/3586)

Unityでマルチスレッドプログラミングを便利にするクラス。System.Threadingのオーバーライド実装みたい。
Update()などメインスレッド以外からでもUnity APIが呼べるのがウリ。けっこう使われてるみたい。たしかに便利。

## Code Example

[http://tsubakit1.hateblo.jp/entry/20121204/1354631826](http://tsubakit1.hateblo.jp/entry/20121204/1354631826) より転載。

```csharp
void Start () {
	thread = new Thread(ThreadTest);
	thread.Start();
}

IEnumerator Createobject() {
	GameObject.Instantiate(obj);
	yield return null;
}

public void ThreadTest() {
	taskFactory.StartNew(Createobject());
	Thread.Sleep(5000);
}	
```

# iterator-tasks

[aiming/iterator-tasks: Iterator Tasks is a iterator-based coroutine class library.](https://github.com/aiming/iterator-tasks)

IteratorベースでCoroutineをまとめて簡潔に書ける。とても簡単なCoruoutineの処理ならこれで十分。

## Code Example

[https://github.com/aiming/iterator-tasks/blob/master/BasicSample/BasicSample.cs](https://github.com/aiming/iterator-tasks/blob/master/BasicSample/BasicSample.cs)

```csharp
using System;
using Aiming.IteratorTasks;

namespace Sample
{
	/// <summary>
	/// 3つのタスクを同時に動かす例。
	/// </summary>
	class BasicSample
	{
		public static void Run()
		{
			var runner = new SampleTaskRunner.TaskRunner();

			Common.ShowFrameTask(50).Start(runner);

			new Task<string>(Worker1)
				.OnComplete(t => Console.WriteLine("Worker 1 Done: " + t.Result))
				.Start(runner);

			new Task<int>(Worker2)
				.OnComplete(t => Console.WriteLine("Worker 2 Done: " + t.Result))
				.Start(runner);

			runner.Update(200);
		}

		/// <summary>
		/// 30フレーム掛けて何かやった体で、文字列を返すコルーチン。
		/// 
		/// 1フレームに処理が集中しないように分割して実行するイメージ。
		/// </summary>
		/// <param name="completed"></param>
		/// <returns></returns>
		private static System.Collections.IEnumerator Worker1(Action<string> completed)
		{
			Console.WriteLine("Start Worker 1");

			for (int i = 0; i < 30; i++)
			{
				yield return null;
			}

			completed("Result");
		}

		/// <summary>
		/// 3秒掛けて何かやった体で、数値を返すコルーチン。
		/// 
		/// スレッドを立ててスリープしている部分を、時間がかかる計算や、ネットワーク待ちに置き換えて考えていただけると。
		/// </summary>
		/// <param name="completed"></param>
		/// <returns></returns>
		private static System.Collections.IEnumerator Worker2(Action<int> completed)
		{
			bool done = false;
			int result = 0;

			System.Threading.ThreadPool.QueueUserWorkItem(state =>
			{
				System.Threading.Thread.Sleep(3000);
				result = 999;
				done = true;
			});

			Console.WriteLine("Start Worker 2");

			while (!done)
				yield return null;

			completed(result);
		}
	}
}
```

# EasyTimer

http://www.dailycoding.com/posts/easytimer__javascript_style_settimeout_and_setinterval_in_c.aspx](http://www.dailycoding.com/posts/easytimer__javascript_style_settimeout_and_setinterval_in_c.aspx)

C#でJSのsetTimeout(), setInterval() 的な処理を実装したもの。Timer.Elapsed イベントをdelegateにして非同期にコールバックメソッドを実行する。
純粋にC#用のstaticクラスだけど、Unity用に書きなおしてみたものが下記。個人的にsetTimeoutやsetIntervalは直感的なのでUnityでもつい使ってみたくなる。

```csharp
using UnityEngine;
using System;
using System.Collections.Generic;
using System.Linq;
using System.Text;

namespace DailyCoding.EasyTimer
{
	public class EasyTimer : MonoBehaviour
	{
		public static IDisposable SetInterval(Action method, int delayInMilliseconds)
		{
			System.Timers.Timer timer = new System.Timers.Timer(delayInMilliseconds);
			timer.Elapsed += (source, e) =>
			{
				method();
			};

			timer.Enabled = true;
			timer.Start();

			// Returns a stop handle which can be used for stopping
			// the timer, if required
			return timer as IDisposable;
		}

		public static IDisposable SetTimeout(Action method, int delayInMilliseconds)
		{
			System.Timers.Timer timer = new System.Timers.Timer(delayInMilliseconds);
			timer.Elapsed += (source, e) =>
			{
				method();
			};

			timer.AutoReset = false;
			timer.Enabled = true;
			timer.Start();

			// Returns a stop handle which can be used for stopping
			// the timer, if required
			return timer as IDisposable;
		}
	}
}
```

## Code Example

```csharp
using UnityEngine;
using UnityEngine.UI;
using DailyCoding.EasyTimer;

public class CorutineTest : MonoBehaviour {

    Start() {

        var stopHandle = EasyTimer.SetTimeout(() => {
                // --- You code here ---
        }, 1000);

        // == clearTimeout
        stopHandle.Dispose();
    }
}
```

# 所感

他にもいろいろありそうですが、とりあえずこれだけ知っとけばだいたいはなんとかなるかなと。  
Unityにも`asyc` `await`はやく来てほしいですねー。
