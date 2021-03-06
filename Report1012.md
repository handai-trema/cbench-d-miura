#課題内容
>## 課題 (Cbench のボトルネック調査)
>
>Ruby のプロファイラで Cbench のボトルネックを解析しよう。
>
>以下に挙げた Ruby のプロファイラのどれかを使い、Cbench や Trema のボトルネック部分を発見し遅い理由を解説してください。
>
>### Ruby 向けのプロファイラ
>
>* [profile](https://docs.ruby-lang.org/ja/2.1.0/library/profile.html)
>* [stackprof](https://github.com/tmm1/stackprof)
>* [ruby-prof](https://github.com/ruby-prof/ruby-prof)
>
>これ以外にもいろいろあるので、好きな物を使ってかまいません。
>
>## 発展課題 (Cbench の高速化)
>
>ボトルネックを改善し Cbench を高速化しよう。

#解答
ruby-profを用いてCbenchのボトルネックを解析した．
Cbenchコントローラの動作を解析するので，

```実行したコマンド
＄ ruby-prof ./bin/trema run ./lib/cbench.rb > output.txt
```

を実行し，結果をテキストファイルとして出力した．
実行結果とプロファイルの出力結果は以下のようになった．
```出力結果
cbench: controller benchmarking tool
   running in mode 'throughput'
   connecting to controller at localhost:6653
   faking 1 switches :: 10 tests each; 10000 ms per test
   with 100000 unique source MACs per switch
   starting test with 1000 ms delay after features_reply
   ignoring first 1 "warmup" and last 0 "cooldown" loops
   debugging info is off
1   switches: fmods/sec:  144   total = 0.014352 per ms
1   switches: fmods/sec:  111   total = 0.011011 per ms
1   switches: fmods/sec:  104   total = 0.010339 per ms
1   switches: fmods/sec:  94   total = 0.009398 per ms
1   switches: fmods/sec:  79   total = 0.007798 per ms
1   switches: fmods/sec:  72   total = 0.007182 per ms
1   switches: fmods/sec:  108   total = 0.010738 per ms
1   switches: fmods/sec:  92   total = 0.009126 per ms
1   switches: fmods/sec:  82   total = 0.008150 per ms
1   switches: fmods/sec:  73   total = 0.007298 per ms
RESULT: 1 switches 9 tests min/max/avg/stdev = 7.18/11.01/9.00/1.39 responses/s
```

```プロファイルの出力(抜粋)
Measure Mode: wall_time
Thread ID: 10766200
Fiber ID: 10764440
Total: 111.723606
Sort by: self_time

 %self      total      self      wait     child     calls  name
  2.48      8.160     2.773     0.000     5.387   488349  *BinData::BasePrimitive#_value
  2.26      3.966     2.524     0.000     1.442   323388   Kernel#define_singleton_method
  2.09      6.847     2.334     0.000     4.513   418709  *BinData::BasePrimitive#snapshot
  1.97      6.313     2.207     0.000     4.107   172427   Kernel#dup
  1.93      2.151     2.151     0.000     0.000   265648   Kernel#initialize_copy
  1.88     22.408     2.101     0.000    20.307   222554   BinData::Struct#instantiate_obj_at
  1.58      5.198     1.768     0.000     3.429   259498   Kernel#clone
  1.57      1.755     1.755     0.000     0.000   207221   BinData::BasePrimitive#initialize_instance
  1.48     23.379     1.657     0.000    21.722   251298   BinData::Base#new
  1.48      5.950     1.655     0.000     4.295   258585  *BinData::BasePrimitive#method_missing
  1.47      1.642     1.642     0.000     0.000   187637   String#size
  1.46      2.980     1.628     0.000     1.352   267975   BinData::SanitizedField#name_as_sym
  1.41     15.112     1.578     0.000    13.534   206288   BinData::Primitive#method_missing
  1.37      1.669     1.525     0.000     0.144   314114   BinData::Base#get_parameter
  1.29      1.442     1.442     0.000     0.000   323388   BasicObject#singleton_method_added
  1.24      1.465     1.390     0.000     0.075   805292   Kernel#respond_to?
  1.22     24.755     1.362     0.000    23.392   251298   BinData::SanitizedPrototype#instantiate
  1.21      1.348     1.348     0.000     0.000   123081   String#unpack
  1.16      1.301     1.301     0.000     0.000   272033   Symbol#to_sym

```

Base::BasePrimitiveやKernelクラスのメソッドに時間をかけていることが分かった．
特に値の参照を行うBinData::BasePrimitive#\_valueの実行に多く時間が取られているので，
cbench.rb内の```match: ExactMatch.new(message)```がボトルネックになっていると考えられる．


[テキスト](http://yasuhito.github.io/trema-book/#_%E7%84%A1%E7%90%86%E3%82%84%E3%82%8A%E9%AB%98%E9%80%9F%E5%8C%96%E3%81%99%E3%82%8B)を参考にし，cbenchプロセスを送り返すFlow Modメッセージを使いまわすことによってcbench.rbの高速化を行った．

cbench.rbを以下の内容に更新し，プロファイルによる解析を再度実行した．

```ruby:cbench.rb
# A simple openflow controller for benchmarking (fast version).
class FastCbench < Trema::Controller
  def start(_args)
    logger.info "#{name} started."
  end

  def packet_in(dpid, packet_in)
    @flow_mod ||= create_flow_mod_binary(packet_in)
    send_message dpid, @flow_mod
  end

  private

  def create_flow_mod_binary(packet_in)
    options = {
      command: :add,
      priority: 0,
      transaction_id: 0,
      idle_timeout: 0,
      hard_timeout: 0,
      buffer_id: packet_in.buffer_id,
      match: ExactMatch.new(packet_in),
      actions: SendOutPort.new(packet_in.in_port + 1)
    }
    FlowMod.new(options).to_binary.tap do |flow_mod|
      def flow_mod.to_binary
        self
      end
    end
  end
end
```
実行結果は以下のようになった

```
cbench: controller benchmarking tool
   running in mode 'throughput'
   connecting to controller at localhost:6653
   faking 1 switches :: 10 tests each; 10000 ms per test
   with 100000 unique source MACs per switch
   starting test with 1000 ms delay after features_reply
   ignoring first 1 "warmup" and last 0 "cooldown" loops
   debugging info is off
1   switches: fmods/sec:  984   total = 0.098276 per ms
1   switches: fmods/sec:  775   total = 0.077471 per ms
1   switches: fmods/sec:  593   total = 0.059285 per ms
1   switches: fmods/sec:  816   total = 0.081598 per ms
1   switches: fmods/sec:  655   total = 0.065398 per ms
1   switches: fmods/sec:  554   total = 0.055340 per ms
1   switches: fmods/sec:  585   total = 0.058483 per ms
1   switches: fmods/sec:  780   total = 0.077953 per ms
1   switches: fmods/sec:  634   total = 0.063338 per ms
1   switches: fmods/sec:  550   total = 0.054909 per ms
RESULT: 1 switches 9 tests min/max/avg/stdev = 54.91/81.60/65.98/9.79 responses/s
```

結果部分を比較すると，高速化前は平均9.00 rsponses/sであったcbenchの応答速度が65.98 responses/sになっており，約７倍高速化することを確認した．
