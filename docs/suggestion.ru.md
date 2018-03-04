## Реализация *type classes* для *FSharp*
1. Синтаксис объявления 
```fsharp
[<Trait>]
type Eq<'t>=
  Equal : 't -> 't -> bool
```
3. Синтаксис реализации:
```fsharp
[<Witness>]
type Eq<int> =
  implement Eq<'t> with
    mamber __.Equal a b = a = b
```
4. Правила имплиментации трейта:
Трейт может быть имплементирован в сборке, где:
- объявлен трейт
- объявлен один из типов, над которым имплементируется трейт
5. Связанные типы трейта.
Описание трейта может включать описание типа:
```fsharp
[<Trait>]
type Iterator<'self> =
  type Item
  abstract Next : 'self -> Item option
  
[<Trait>]
type IntoIterator<'self> =
  type Item
  toIterator : 'self -> Iteraror<Item = Item> 
```
```fsharp
[<Withess>]
type Iterator<IEnumerator<'t>> =
  interface Iterator<IEnumerator<'t>, Item = 't> with
    member __.Next en =  if en.Next() then en.Current |> Some else None

[<Withess>]
type IntoIterator<List<'t>> =
  interface IntoItrator<List<'t>, Item = 't> with
    member __.toIterator lst = IEnumerable.GetEnumerator<'t> lst 
  
```

6. Компиляция трейтов:
Компилируются в интерфейсы:
```fsharp
type Eq<'self> =
  abstract Equal : 'self -> 'self -> bool
```
```fsharp
type Iterator<'self, 'item> =
  abstract Next: 'self -> 'item
type IntoIterator<'self, 'item , 'iter when 'iter :> Iterator<'iter, 'item>> =
  
  abstract toIterator: 'self -> 'iter
```
7. Компиляция реализаций:
Компилируются в структуры или расширения:
```fsharp
[<Struct>]
type EqInt =
  interface Eq<int> with
    member __.Equal a b  = a = b
```    
```fsharp
type IEnumerator<'t> with
  interface Iterator<IEnumerator<'t>, 't> with // это то же расширение тогда, надо в inline запихивать чтобы сейчас компилилось. Но проще впрямую в комптлер вкатать
    member __.Next en =  if en.Next() then en.Current |> Some else None
[<Struct>]    
type IntoIterator<List<'t>, IEnumerator<'t>, 't> =
  interface IntoIterator<List<'t>, IEnumerator<'t>, 't> with
    member __.toIterator lst = IEnumerable.GetEnumerator<'t> lst
```
8. Добавляется атрибутная разметка либо к типу, либо к реализации:
```fsharp
[<Struct>]
[<TraitImpl(typeof(Eq<Int>))>]
type EqInt =
  interface Eq<int> with
    member __.Equal a b  = a = b
```    
9. Правка выводфа типов с учетом атрибутов
10. Runtime поиск имлементации по типу атрибуту с кешированием:
~~~fsharp
  let tc = witness<Eq<int>>
  tc.Eq 7 8
  let tc = witness<Eq> 5
  tc.Eq 4
  let it = witness<IntoIterator<List<'T>>> 
  for i in it.toIterator([1, 3, 5]) do
    printf "%A" i
  let it = witness<IntoIterator> [1, 3, 6]
  for i in it.toIterator() do
    printf "%A" i
~~~


11. Пример реального использования:

Сейчас:
```fsharp
open System
open System.Collections.Generic
open System.Collections.Immutable

type SToKeyValuePair<'t, 'k, 'v> =
    abstract ToPair : 't -> KeyValuePair<'k, 'v>
    
[<Struct>]
type KV<'k, 'v> =
    interface SToKeyValuePair<KeyValuePair<'k, 'v>, 'k, 'v> with
            member __.ToPair t = t 
    
[<Struct>]
type KVTuple<'k, 'v> =
    interface SToKeyValuePair<Tuple<'k, 'v>, 'k, 'v> with
        member __.ToPair ((k, v)) = KeyValuePair(k, v)
        
[<Struct>]
type KVSTuple<'k, 'v> =
    interface SToKeyValuePair<ValueTuple<'k, 'v>, 'k, 'v> with
        member __.ToPair (struct (k, v)) = KeyValuePair(k, v)  
        
     
type HashMap<'key, 'value> = ImmutableDictionary<'key, 'value>

module HashMap =
    open System.Collections.Generic

    let private toKvp = fun (k, v) -> new KeyValuePair<'key, 'value>(k, v)
    let private toKvpS = fun struct (k, v) -> new KeyValuePair<'key, 'value>(k, v)
    
    let ofSeq<'tc, 't, 'k, 'v when 'tc :> SToKeyValuePair<'t, 'k, 'v> and 'tc : struct> sq  : HashMap<_, _> =
        let tc = Unchecked.defaultof<'tc>
        ImmutableDictionary.CreateRange(sq |> Seq.map tc.ToPair)     
    let ofKeyValuePairs sq = ofSeq<KV<_, _>,_, _, _> sq
    let ofTuples sq  = ofSeq<KVTuple<_, _>, _, _, _> sq
    let ofValueTyples sq = ofSeq<KVSTuple<_, _>, _, _, _> sq
    
    let set k v (map : HashMap<'key, 'value>) : HashMap<_, _> = map.SetItem(k, v)
    let set2<'tc, 't, 'k, 'v when 'tc :> SToKeyValuePair<'t, 'k, 'v> and 'tc : struct> v map =
       let tc = Unchecked.defaultof<'tc>
       let p = tc.ToPair(v)
       set p.Key p.Value map  
    let setKeyValuePair p map  = set2<KV<_, _>, _, _, _> p map
    let setTuple p map  = set2<KVTuple<_, _>, _, _, _> p map
    let setValueTuple p map  = set2<KVSTuple<_, _>, _, _, _> p map
     
    let add k v (map : HashMap<'key, 'value>) : HashMap<_, _> = map.Add(k, v)
    let add2<'tc, 't, 'k, 'v when 'tc :> SToKeyValuePair<'t, 'k, 'v> and 'tc : struct> v map =
        let tc = Unchecked.defaultof<'tc>
        let p = tc.ToPair(v)
        add p.Key p.Value map
    let addKeyValuePair p map  = add2<KV<_, _>, _, _, _> p map
    let addTuple p map  = add2<KVTuple<_, _>, _, _, _> p map
    let addValueTuple p map  = add2<KVSTuple<_, _>, _, _, _> p map
    
    let setItems<'tc, 't, 'k, 'v when 'tc :> SToKeyValuePair<'t, 'k, 'v> and 'tc : struct> sq (map : HashMap<_, _>) : HashMap<_, _> =
       let tc = Unchecked.defaultof<'tc>
       map.SetItems(sq |> Seq.map tc.ToPair)     
    let setKeyValuePairs sq map  =setItems<KV<_, _>, _, _, _> sq map
    let setTuples sq map  =setItems<KVTuple<_, _>, _, _, _> sq map
    let setValueTuples sq map  =setItems<KVSTuple<_, _>, _, _, _> sq map
    
    let addRange<'tc, 't, 'k, 'v when 'tc :> SToKeyValuePair<'t, 'k, 'v> and 'tc : struct> sq (map : HashMap<_, _>) : HashMap<_, _> =
           let tc = Unchecked.defaultof<'tc>
           map.AddRange(sq |> Seq.map tc.ToPair)
    let addKeyValuePairs sq map  = addRange<KV<_, _>, _, _, _> sq map
    let addTuples sq map  = addRange<KVTuple<_, _>, _, _, _> sq map
    let addValueTuples sq map  =addRange<KVSTuple<_, _>, _, _, _> sq map
    
    let removeAll k (map : HashMap<'key, 'value>) = map.RemoveRange(k)
```
После:
```fsharp
open System
open System.Collections.Generic
open System.Collections.Immutable

[<Trait>]
type ToKeyValuePair<'t, 'k, 'v> =
    ToPair : 't -> KeyValuePair<'k, 'v>

[<Witness>]
type ToKeyValuePair<KeyValuePair<'k, 'v>, 'k, 'v> = 
  interface ToKeyValuePair<KeyValuePair<'k, 'v>, 'k, 'v> with
    member __.ToPair t = t 
    
[<Witness>]
type ToKeyValuePair<'k * 'v,'k, 'v> =
  interface ToKeyValuePair<'k * 'v, 'k, 'v> with
    ToPair ((k, v)) = KeyValuePair(k, v)
        
[<Witness>]        
type ToKeyValuePair<struct 'k * 'v,'k, 'v> =
  interface ToKeyValuePair<struct 'k * 'v, 'k, 'v> with
    ToPair (struct (k, v)) = KeyValuePair(k, v)
     
type HashMap<'key, 'value> = ImmutableDictionary<'key, 'value>

module HashMap =
    open System.Collections.Generic

    let ofSeq (sq : ToKeyValuePair seq)  : HashMap<_, _> =
        ImmutableDictionary.CreateRange(sq |> Seq.map tc.ToPair)     
    
    let set k v (map : HashMap<_, _>) : HashMap<_, _> = map.SetItem(k, v)
    let setPair (p: ToKeyValuePair) map  = let v = p.ToPair() in set v.Key v.Value map
        
    let add k v (map : HashMap<_, _>) : HashMap<_, _> = map.Add(k, v)
    let addPair (p: ToKeyValuePair) map  = let v = p.ToPair() in add v.Key v.Value map
    
    let setItems (sq : ToKeyValuePair seq) (map : HashMap<_, _>) : HashMap<_, _> =
        map.SetItems(sq |> Seq.map tc.ToPair)     
    
    let addRange(sq : ToKeyValuePair seq) (map : HashMap<_, _>) : HashMap<_, _> =
        map.AddRange(sq |> Seq.map tc.ToPair)
    
    let removeAll k (map : HashMap<_, _>) = map.RemoveRange(k)
```
