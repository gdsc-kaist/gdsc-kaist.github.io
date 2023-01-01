
# Range

- 참고 자료: Rust Python github repository, range.rs
[RustPython/range.rs at 5fed6a35af2f1ffec4e6906df5d7d59b24f7d147 · RustPython/RustPython](https://github.com/RustPython/RustPython/blob/5fed6a35af2f1ffec4e6906df5d7d59b24f7d147/vm/src/builtins/range.rs)

## 목표
Rust Python의 range 파이썬 빌트인 클래스 구현 코드를 살펴봄으로써 파이썬 빌트인 함수가 어떤 식으로 구동되는지 알아 본다.


## Python Built-in class Range에 대한 이해

파이썬 Documentation에 본 게시물을 이해하기 위해서 필요한 range 클래스에 대한 사전 정보가 잘 정리되어 있으니, 본 글을 더 잘 이해하고 싶다면 아래 링크를 타고 들어가 range에 대한 설명을 읽어 보기를 바란다.

[Built-in Types](https://docs.python.org/3/library/stdtypes.html#range)

Range 클래스는 start, stop, step 세 가지 변수로만 구성되어 있는 sequence입니다. range 클래스의 특징을 간단히 알아 본다.


### Range type의 특징

1. **immutable sequence of numbers이다.**

range 타입은 number sequence이다. range 타입 변수 r이 있다고 할 때, r[i]=start+i*step으로 계산된다. 이때 i는 0 이상의 정수여야 한다. 만약 step > 0이라면 r[i] < stop이어야 하고, step < 0이면 r[i] > stop이어야 한다. step == 0인 경우에는 range는 정의되지 않는다. 이때 start, stop, step은 모두 정수여야 한다.

2. **Python의 Sequence ADT를 구현한다. 따라서 containment test, indexing같은 sequence ADT에서 사용할 수 있는 메소드를 구현한다. 하지만 concaternation, repetition 등은 구현하지 않는다.**

정확히 말하면 range는 [collections.abc.Sequence](https://docs.python.org/3/library/collections.abc.html#collections.abc.Sequence) 를 구현한다. sequence 타입에는 containment test, indexing, slicing 등의 기본적인 메소드들이 존재한다. 그러나 range는 repetition과 concaternation 메소드를 구현하지 않는다. range는 등차순열 패턴을 이용해 수열을 생성하는 타입인데, repetition이나 concatenation 메소드를 range 타입에 적용하면 그 패턴이 깨지기 때문이다.

sequence abstract base class에 어떤 메소드들이 구현되어 있는지는 다음 python documentation을 참고하도록 한다.


3. **list 타입과 달리, range sequence의 요소들을 메모리에 다 저장해두는 것이 아니라, 필요한 r[i] 값이나 subsequence의 값을 필요한 순간에 계산하는 방식을 쓴다. 따라서 메모리 효율적이다.**

list 타입은 list 생성과 동시에 필요한 primitive value를 메모리에 생성하고 그것에 대한 레퍼런스를 메모리에 저장한다. 반면 range 타입은 인스턴스에 start, stop, step만 저장하고 필요한 value나 substring을 메소드 호출 시에 필요에 따라 계산하는 방식으로 구현되어 있다. 따라서 range sequence의 길이에 관계 없이 range 인스턴스가 메모리에서 차지하는 크기는 매우 작고, 고정되어 있으며 따라서 list에 비해 메모리 효율적이다.

## Rust Implementation 살펴 보기

 이제 python range를 얼마 정도 살펴 봤으니 rust에 python range 타입이 어떻게 구현되어 있는지 살펴 보자.

pyrange 클래스는 다음과 같이 python integer type의 start, stop, step만을 저장한다.

```rust
pub struct PyRange {
    pub start: PyIntRef,
    pub stop: PyIntRef,
    pub step: PyIntRef,
}
```

아래 부분은 sequence나 iterator 등의 ADT의 메소드가 아니라 range 자체의 메소드를 구현하는 부분이다. 코드가 쉽게 작성되어 있으므로 메소드 몇 개만 간단히 살펴 보고 넘어 가자.

- indexof()

```rust
pub fn index_of(&self, value: &BigInt) -> Option<BigInt> {
	let step = self.step.as_bigint();
	match self.offset(value) {
		Some(ref offset) if offset.is_multiple_of(step) => Some((offset / step).abs()),
// (1) valid한 value였고 (2) offset이 step의 multiple이었을 경우, index값을 반환함.
		Some(_) | None => None, // invalid한 value였을 경우 None을 Return.
}
```

앞서 계산한 offset 값을 이용하여 value가 range sequence 내의 합당한 원소인지, 합당한 원소라면 index value를 계산해서 리턴한다. 만약 합당한 원소가 아니면 none을 반환한다.

- get()
```rust
pub fn get(&self, index: &BigInt) -> Option<BigInt> {
	let start = self.start.as_bigint();
	let step = self.step.as_bigint();
	let stop = self.stop.as_bigint();
	
	if self.is_empty() {
		return None;
	}
	if index.is_negative() {
		let length = self.compute_length();
		let index: BigInt = &length + index; // start로부터의 index 계산
		if index.is_negative() {
		return None; // index가 범위 벗어남.
	}
	Some(if step.is_one() {
		start + index
	} else {
		start + step * index
	}) // 알맞은 값을 리턴함.
	//posive index에 대해 똑같은 동작 수행.
	} else {
		let index = if step.is_one() {
		start + index
	} else {
		start + step * index
	};
	if (step.is_positive() && stop > &index) || (step.is_negative() && stop < &index) {
		Some(index)
	} else {
		None
	}
}
```
 python integer type index i를 input으로 넣으면 output으로 range r의 r[i]를 반환하는 함수이다. r 자체가 비어있는 경우와, i >= 0인 경우, i<0를 구별하여 처리한다. i가 양수인 경우와 음수인 경우 함수의 동작은 똑같으니 i가 음수인 경우만 살펴 보자. 우선, r 자기 자신의 길이와 index를 비교해서, index의 크기가 length를 벗어나는 out of bound case인지를 검사한다. 만약 out of bound라면 None을 반환한다. 아니라면  start+step *i를 통해 r[i]를 반환한다.



이제 다른 부분으로 넘어 가자. 이 부분 이후 Sequence, Mapping, Hashable, Comparable, Iterable 타입의 Abstract Base Class의 메소드를 구현하는데, 대부분의 메소드는 이해하기 쉽고 자명하게 구현되어 있어 rust 문법만 잘 알고 있다면 대부분 쉽게 이해할 수 있을 것이다. 만약 이해가지 않는 부분이 있다면 아마도 아래 설명에 나오는 부분과 virtual machine (VM)과 관련된 부분일 것이다.

이 코드는 277번째 라인의 contains 메소드이다.

```rust
fn contains(&self, needle: PyObjectRef, vm: &VirtualMachine) -> bool {
        // Only accept ints, not subclasses.
        if let Some(int) = needle.payload_if_exact::<PyInt>(vm) {
            match self.offset(int.as_bigint()) {
                Some(ref offset) => offset.is_multiple_of(self.step.as_bigint()),
                None => false,
            }
        } else { // needle이 int로 안 까진 경우여서 offset function에 먹일 수 없는 경우인가봄.
            // 굳이 이렇게 iterative search를 할 필요가 있는건가? 그냥 계산하면 될 것 같은데. python의 dynamic type 때문에 이런 iter search가 필요함.
            iter_search(
                self.clone().into_pyobject(vm),
                needle,
                SearchType::Contains,
                vm,
            )
            .unwrap_or(0)
                != 0
        }
    }
```

코드를 보면 needle 즉 찾고자 하는 숫자가 integer type이면 offset 계산한 후 인덱스 계산해서 range 시퀀스에 포함되는지 계산한 후 적절한 값을 리턴한다. 그러나 python integer type이 아니라면, [range.rs](http://range.rs) 제일 처음에 정의되는 iter_search() 함수를 이용하여 range 오브젝트의 값을 하나하나 needle과 비교하는 exhaustive search를 시도한다. 이 부분이 굳이 있을 필요가 있나 싶기도 하지만, python은 dynamic type을 지원하기 때문에 가끔 Range 안에 2.0과 같은 number를 넣어도 알아서 integer type으로 처리해서 함수값을 내어줘야 한다. 이때문에 이러한 iter_search가 사용된 것으로 보인다.

 다른 많은 메소드에서도 이런 부분이 많은데, 대부분 dynamic type으로 integer로 처리될 수 있는 non-py integer type number가 함수에 사용된 경우를 처리하기 위한 방법으로 생각하면 된다.

아마도 이 부분과 virtual machine과 관련된 부분을 제외한다면 코드를 이해하기 어렵지 않았을 것 같다. virtual machine과 관련된 부분은 다른 게시물을 찾아서 이해하는 것을 추천한다.

 코드가 쉽게 적혀 있어 코드의 모든 부분을 같이 읽으며 하나하나 해설하는 것은 필요 없을 것 같다. 이번 게시물에서 python range.rs를 살펴보며 다음 내용을 알고 갔으면 좋겠다.

- Range 타입과 list 타입 둘 다 sequence를 구현하는데, 둘의 차이점은 무엇인지? (메모리 관점)
- 위에서 본 range 타입의 구현방식으로 인하여 sequence나 hashable 등 다른 ABC의 메소드에 대한 구현 방식은 어떻게 영향을 받는지
- Virtual machine과 관련하여 파이썬 오브젝트는 rust 내에서 어떻게 상호 소통하고 동작하는지. 이 내용은 python vm in rust python을 먼저 이해하고 살펴보면 좋겠다.