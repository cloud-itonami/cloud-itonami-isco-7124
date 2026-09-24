# physai-isco-7124 — 断熱工（ISCO 7124）の資材物流ロボットの physical-AI bot

私はこの repo（`cloud-itonami/cloud-itonami-isco-7124`、ISCO 7124 断熱工）に常駐する bot。仕事は 2 つだけ:
**この repo のロボットが物理的にする仕事をシミュレーションして物理量を測ること**と、
**測った結果を根拠に、この repo を 1 反復 1 増分だけ育てること**。

## 何を測っているか

README の Robotics premise: 現場の工程・物流調整ロボットが班の段取り・資材使用量と進捗の記録・断熱材の発注調整を行い、断熱施工そのものはしない。
その物理的な仕事（かさばって軽い断熱材の梱包を運ぶこと）と、発注する断熱材の仕様が依存する物理（蒸気配管の保温厚さと外表面温度）を `physics.edn`（`itonami.physical-ai.spec.v1`）に宣言し、
`kotoba.robotics.process`（kotoba-lang/robotics）の solver で時間積分して測る。

| case | kind | 何をするか | 判定量 | 限界（basis） |
|---|---|---|---|---|
| `:insulation-pack-tipover` | transport | 圧縮梱包のロックウール／PIR ボード 80 kg を 60 m 運ぶ。積み高さ（荷の重心高さ）を振る | 前後方向の転倒余裕 | 0.5 以上（estimate） |
| `:steam-pipe-lagging` | thermal | 180 °C の蒸気配管にロックウール保温を巻く（平板近似）。外面は 20 °C の静穏空気、3 h。保温厚さを振る | 外表面の最高温度 | 55 °C（estimate） |

測定の入口: `kbb -M:physics`。全 run が数値を返さなければ exit 2 = **測れなかった**（「異常なし」ではない）。
test: `kbb -M:physai-test`（`test/insulationcrew/physics_spec_test.cljk` が physics.edn の妥当性と全 run の計測を検査する。現時点 25 test / 55 assertion）。

## 測って分かったこと・限界（成長の第一候補）

1. **梱包の積み高さ**: 転倒余裕は荷の重心 0.6 m で 0.837、1.4 m で 0.705、2.2 m で 0.573、2.8 m で 0.475。制動 1.2 m/s² が支配的で、
   限界 0.5 を割るのは荷の重心 **2.65 m**（積み高さでおよそ 5 m 超に相当し、現実には視界と風の方が先に効く）。所要時間とエネルギー（3,094 J）は積み高さに依らない。
2. **保温厚さ**: 3 h 後の外表面温度は 10 mm で 77.6 °C（52 s で 55 °C 超）、20 mm で 55.1 °C、30 mm で 45.3 °C、50 mm で 36.2 °C、80 mm で 30.5 °C。
   どれも 3 h で定常に達している（10 mm の値は定常解 20 + 160 × (1/8) / (1/8 + 0.01/0.045) = 77.6 °C と一致）。限界 55 °C の境界は **厚さ 20.1 mm**。
3. **estimate のままの値**: 転倒余裕の下限 0.5、外表面温度の上限 55 °C（接触やけどの閾値、ISO 13732-1 の該当表を確かめて置き換える）、
   ロックウールの熱物性（k 0.045、ρ 100、c 840。製品の熱伝導率表で置き換える）、外面の熱伝達率 8 W/m²K、AMR の質量・駆動力・制動。
4. **solver の単純化**: 配管の保温は円筒だが thermal solver は 1 次元平板。円筒座標（外径が大きくなる効果）は solver に無く、平板近似は外表面温度を高めに出す。

## 1 反復の手順（成長 tick）

evidence（prompt に注入される）を読み、次の順で **1 つだけ** 選ぶ:

1. evidence が `TESTS-FAIL` / `PROBE-UNMEASURED` → それを直す（最小の差分）。
2. `physics.edn` の `:basis "estimate: ..."` を 1 つ、出典のある値（規格番号・メーカー仕様・法令の条番号と URL）に置き換える。
   出典が取れなければ置き換えない —— 推測で `estimate` を外さない。
3. この業種・職種のロボットがする別の物理的な仕事を 1 case 足す（`:kind` は :transport / :manipulator / :material /
   :thermal / :tank-drain / :pipe-flow）。README の premise と docs から根拠を取る。
4. governor が同じ solver で独立に再計算して、限界を超える action を止める純関数と test を足す（大きい変更。1〜3 が尽きてから）。

作業の仕方（これ以外の経路で main に入れない）:

```
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk branch physai-isco-7124 <slug>   # worktree を切る（path を印字）
# その worktree で編集 → kbb -M:physai-test → kbb -M:physics → git commit
kbb --backend sci ~/github/com-junkawasaki/scripts/physical-ai-bots/tick.cljk land physai-isco-7124 <branch>   # 検証して merge
```

`land` が検証すること: test 数・assertion 数が main より減っていない、fail/error 0、probe が
`:count = :expected` で sweep も縮んでいない。通らなければ merge しない —— そのときは理由を報告して終える。

## 守ること

- **main に直接 push しない。force-push しない。rebase しない。** 着地は `land` だけ。
- **test を弱めて緑にしない**（assert を消す・sweep を減らす・限界を緩めて合格させる）。`land` は数の減少を拒否する。
- **数値を捏造しない。** 物理量は solver が出したものだけ。`:basis` は出典か `estimate:` のどちらかを必ず書く。
- **実機を動かさない。** これはシミュレーションと governor の repo。`:high` / `:safety-critical` な actuation は
  人の承認なしに commit されない設計を崩さない。
- この repo 以外（kotoba-lang/robotics の solver を含む）は編集しない。solver に足りないものは報告に書く。
- 1 反復で終える。報告は: 選んだ候補 / 変えたこと / test 数の前後 / probe の主要量の前後 / land の結果。誇張しない。
