# 本書について

本書は、私が RDSR について調べたことを、私の視点でまとめたものです。
主に、RDSRのDICOMフォーマットでの保存形式について調べたため、対象は
RDSRというよりも、SR(Structured Report)についての調査結果のほうが
近いかもしれません。

ただし、あくまでもターゲットはRDSRなので、RDSRでは現れない形式についての
調査はほぼ全く行っていいません。

# 対象読者

対象読者は、以下を想定しています。

1. DICOM の基本的な知識はある人。
   PS3.3のIODや、PS3.5のDICOMフォーマットについての基礎的な知識は前提としています。
2. 規格書を呼んでもRDSRの記載から、どのようにDICOMのフォーマットに展開されるのかがイメージできない人。
   結局は、規格書に書いてあることを私の言葉で解説しているだけです。

# DICOM SRの構造の概要

ターゲットはRDSRですが、フォーマットについての話なので、最初はDICOM SR全般の話から入ります。

DICOM SRを説明する場合、タグの構造の話と、テンプレートの話があります。

個人的な感触ですが、世の中の解説書はテンプレートについての説明が大半で、
タグ構造について説明されているものはほとんど見たことがありません。

*タグ構造* : データのバイナリストリームにどうやって表現するかの規則。     
*テンプレート構造* : どのようなデータをどのような構造で保持・管理するかについて、概念的な構造の規則。

これ以降、しばらくはDICOM SRのタグ構造について述べていきます。

# DICOM SRのタグ構造

## 対応するDICOM規格

PS3.3 A35.8(X-Ray Radiation Dose SR IOD)から始まって、C17.3 (SR Document Content) とたどります。

この、 C17.3 (SR Document Content) がRDSRのテンプレート構造をいかにDICOMのタグ構造で表すかを説明している部分です。

ただ、C17.3だけ見ても何のことだかさっぱりとなると思います。

この規格書の記載だで十分理解できた人は、これ以上私の解説を読む必要はありません。

ちなみに、ここまで読んだ時の私の疑問は以下のようなものでした。

- よく、SRの構造ってXMLと似ているって言われているけど、これのどこがXMLなんだ?
- XMLのタグやテキストとこのタグ構造って、一体どう対応するんだ?

## XMLでの表現

XMLとDICOM SRの構造の対応は、トップダウンよりもボトムアップのほうが理解しやすいと思います。

ですから、最初は比較的小さい塊を考えることにします。例として、1回のばく射実績を表す組(撮影管電圧=80kV, 管電流=200mA, 撮影時間=5ms)を表現したいとします。

XMLで表現するとなると、例えば、以下のような表現が考えられます。

    <IrradiationEventXRayData>
        <KVP unit="kV">80</KVP>
        <XRayTubeCurrent unit="mA">200</XRayTubeCurrent>
        <ExposureTime unit="msec">5<ExposureTime>
    </IrradiationEventXRayData>

RDSRを調べたことのある人なら想像できると思いますが、
PS3.16 (Appendix A. Structured Reporting Templates(Normative))の、以下2つのテンプレートから説明用に一部を抜粋したものになっています。

* TID 10003 Irradiation Event X-Ray Data ←親要素
* TID 10003B Irradiation Event X-Ray Source Data ←子要素

タグの名前は、TID 10003 および TID 10003B に倣っています。

> なぜ、わざわざ親要素の名前だけ TID10003 から取ってきたのか。
> 
> それは、TID 10003 の先頭が CONTAINER となっており、
> それ以降の行は1階層下の扱いになっています。
> TID 10003 の中で、TID 10003B をINCLUDE する格好になっていますが、
> TID 10003B では階層がつけられていません。
> これは、TID 10003B の各要素は、親のTID 10003 から階層を付けずに
> INCLUDEされることになります。
> 
> 今回の説明の主役は子要素ですが、親要素については一階層上のものを
> 選びたかったからです。

これをDICOM SRの構造に変換するために、以下のように置き換えていきます。

## 無機質なタグ表現への置き換え

先のXMLは、タグ名が意味を持ってしまっています。
このタグ名を無機質なものにします。

### 子要素の置き換え

まず、子要素を無機質化します。

- 各項目について、key-value を持つ、node で表現することにします。
  各子要素は、項目を表す&lt;node&gt; タグの配列に置き換えられます。
- &lt;node&gt;タグには、その型を表す属性を付けます。TID 10003B のVT列を参照すると、
  "NUM"と記載されているので、VT="NUM" のように属性を付けておきます。
- &lt;node&gt; タグの子供には、key を表す &lt;key&gt; タグ,
  value を表す &lt;num_value&gt; タグが含まれます。    
  また、&lt;num_value&gt; タグには、元のタグにあった単位をそのままつけておきます。    
  (value でなく、num_value にしたのは、VT="NUM"だからです。あとで出てきますが、VT="CONTAINER" の場合、content_value にします)


これらの子要素は、TID 1003B の対応する行のVT列を見ると、"NUM"と記載されています。

ここまでの置き換えを行うと以下のようなXML表現になります。

    <IrradiationEventXRayData>
        <node VT="NUM">
            <key>KVP</key>
            <num_value unit="kV">80</num_value>
        </node>
        <node VT="NUM">
            <key>XRayTubeCurrent</key>
            <num_value unit="mA">200</num_value>
        </node>
        <node VT="NUM">
            <key>ExposureTime</key>
            <num_value unit="ms">5</num_value>
        </node>
    </IrradiationEventXRayData>

1項目1行で表現されてた者が、4行に膨れ上がっています。

### 親要素の置き換え

この段階では、親タグはまだ"IrradiationEventXRayData"という、
意味を持ったタグになっています。
これも、無機質な表現にしてしまいます。

- 子要素と同じように、親要素も、&lt;node&gt; タグに置き換えます。
- ただ、こちらはVTが”CONTAINER"なので、VT属性は、"CONTAINER"にします。
- 子要素の複数の &lt;node&gt; を表すため、&lt;contents&gt;で括ります。
  その結果、子要素の各 &lt;node&gt; は、一階層分ネストされます。

親要素も置き換えを行うと、以下のようなXML表現になります。

    <node VT="CONTAINER">
        <key>IrradiationEventXRayData</key>
        <content_value>
            <node VT="NUM">
                <key>KVP</key>
                <num_value unit="kV">80</num_value>
            </node>
            <node VT="NUM">
                <key>XRayTubeCurrent</key>
                <num_value unit="mA">200</num_value_>
            </node>
            <node VT="NUM">
                <key>ExposureTime</key>
                <num_value unit="ms">5</num_value>
            </node>
        </content_value>
    </node>

元のXMLと比較すると以下のようなことが分かります。

- 全てのタグが、&lt;node&gt;で置き換えられる。
- 元のタグ名は、&gt;node&gt;の子要素の&lt;key&gt;タグで表現される。
- 元の値は、&lt;node&gt;の子要素の&lt;xxx_value&gt;タグで表現される。
- 階層構造は、親要素の&lt;content_value&gt;の下に再度 &lt;node&gt;が連なる格好になる。

この結果、  

> &lt;node(親要素)&gt;-&lt;content_value&gt;
> -&lt;node(子要素)&gt;-&lt;xxx_value&gt;

のようになり、元々2階層で表現されていたものが、4階層になります。




