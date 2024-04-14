## HashMap과 HashTable

==Hashtable== Java 1부터 존재한 클래스입니다. 그리고 HashMap은 Java 2부터 나온 클래스입니다. Map 인터페이스를 구현하고 있기 때문에 제공하는 기능은 같습니다.

차이점은 아래와 같습니다.

- HashMap은 보조해시를 사용하기 때문에 보조해시 함수를 사용하지 않는 Hashtable에 비하여 해시 충돌이 덜 발생할 수 있어 상대적으로 성능상 이점이 있습니다.
- HashMap은 Thread Safe하지 않고, Hashtable은 Thread Safe 합니다.
- HashMap은 지속적으로 개선 중에 있습니다.

### Hash 분포와 Hash 충돌

HashMap은 버킷의 위치를 정할 때 객체의 해시코드를 사용합니다. 이 때 해시코드의 결과 자료형은 int입니다.
int로는 완전한 Hash 함수를 만들 수 없습니다. 이유는 그 크기 보다 더 많은 객체를 생성할 수 있기 때문입니다. 또한 그만한 버킷을 만드는 것은 엄청난 메모리의 낭비이기 때문입니다. 
X라는 데이터를 사이즈가 M인 HashMap에 put할 때 index는 아래와 같이 구할 수 있습니다.
```
int hash = X.hashCode() % M;
```

위와 같이 나눠주면 충돌이 발생하지만 표현해야할 모든 범위만큼의 버킷은 가지지 않아도 됩니다.
![Pasted image 20240414115649](https://github.com/inturn86/TIL/assets/110794550/190d0618-7aef-450f-bd2b-25a17be201a0)




그렇다면 Hash 충돌이 일어났을 경우 어떤식으로 처리가 될까요? 충돌을 해결하기 위해 개방 주소법(Open Addressing)과 분리 연결법(Seperate Chaining)을 사용합니다.

#### 개방 주소법(Open Addressing)

개방 주소법(Open Addressing)이란 추가적인 메모리를 사용하는 Chaining 방식과 다르게 비어있는 해시 테이블의 공간을 활용하는 방법입니다. `개방 주소법`을 구현하기 위한 대표적인 3가지 방식이 존재합니다.

1. `Linear Probing`: 만약 충돌이 h[k]에서 난다면 h[k + 1]이 비어있는 확인하고 비어 있지 않다면 h[k + 2] . . . 식으로 계속 확인하는 방법입니다.
2. `Quadratic Probing`: 해시의 저장순서 폭을 제곱으로 저장하는 방식입니다. 예를 들어 처음 충돌이 발생한 경우에는 1만큼 이동하고 그 다음 계속 충돌이 발생하면 2^2, 3^2 칸씩 옮기는 방식입니다.
3. `Double Hashing Probing`: 해시된 값을 한번 더 해싱하여 새로운 주소를 할당하기 때문에 다른 방법들보다 많은 연산을 하게 됩니다.

1, 2번은 버킷 조사는 원형으로 회전하게 됩니다. 테이블의 마지막에 도달하면 다시 처음으로 이동합니다.

![Pasted image 20240414115635](https://github.com/inturn86/TIL/assets/110794550/e59f35c0-504e-44bc-badd-a45e5186995d)

#### 분리 연결법(Seperate Chaining)

Java HashMap에서도 이용하고 있는 방식입니다. Separate Chaining이란 동일한 버킷의 데이터에 대해 리스트 or 트리 자료구조를 이용해서 추가 메모리를 사용하여 다음 데이터의 주소를 저장하는 것입니다. 아래의 그림과 같이 충돌이 발생했을 때 다음 노드를 연결하고 있는 것을 볼 수 있습니다.

그리고 충돌이 많이 발생해서 리스트의 형태로 계속 데이터가 쌓이게 되면 검색하는데 시간 복잡도가 O(n)으로 나빠지게 됩니다. 그래서 Java8의 HashMap은 리스트의 개수가 8개 이상이 되면 `Self-Balancing Binary Search Tree` 자료구조를 사용해 Chaining 방식을 구현하였습니다. (탐색할 때 O(logN)으로 성능이 좋아집니다.)

![Pasted image 20240414115758](https://github.com/inturn86/TIL/assets/110794550/34fac056-9e8e-49f7-886d-fc52cfaa7e82)
### Java 7에서의 해시 버킷 관련 구현

```java
transient Entry<K,V>[] table = (Entry<K,V>[]) EMPTY_TABLE;  
// transient로 선언된 이유는 직렬화(serializ)할 때 전체, table 배열 자체를 직렬화하는 것보다
// 키-값 쌍을 차례로 기록하는 것이 더 효율적이기 때문이다.


static class Entry<K,V> implements Map.Entry<K,V> {  
        final K key;
        V value;
        Entry<K,V> next;
        int hash;

		Entry(int h, K k, V v, Entry<K,V> n) {  
            value = v;
            next = n;
            key = k;
            hash = h;
        }

        public final K getKey() { … }
        public final V getValue() { …}  
        public final V setValue(V newValue) { … }
        public final boolean equals(Object o) { … }
        public final int hashCode() {…}
        public final String toString() { …}

		void recordAccess(HashMap<K,V> m) {… }

		void recordRemoval(HashMap<K,V> m) {…}  
}
```

```java
public V put(K key, V value) { if (table == EMPTY_TABLE) { inflateTable(threshold); // table 배열 생성 } // HashMap에서는 null을 키로 사용할 수 있다. if (key == null) return putForNullKey(value); // value.hashCode() 메서드를 사용하는 것이 아니라, 보조 해시 함수를 이용하여 // 변형된 해시 함수를 사용한다. "보조 해시 함수" 단락에서 설명한다.  
        int hash = hash(key);

        // i 값이 해시 버킷의 인덱스이다.
        // indexFor() 메서드는 hash % table.length와 같은 의도의 메서드다.
        int i = indexFor(hash, table.length);

        // 해시 버킷에 있는 링크드 리스트를 순회한다.
        // 만약 같은 키가 이미 저장되어 있다면 교체한다.
        for (Entry<K,V> e = table[i]; e != null; e = e.next) {
            Object k;
            if (e.hash == hash && ((k = e.key) == key || key.equals(k))) {
                V oldValue = e.value;
                e.value = value;
                e.recordAccess(this);
                return oldValue;
            }
        }

        // 삽입, 삭제 등으로 이 HashMap 객체가 몇 번이나 변경(modification)되었는지
        // 관리하기 위한 코드다.
        // ConcurrentModificationException를 발생시켜야 하는지 판단할 때 사용한다.
        modCount++;

        // 아직 해당 키-값 쌍 데이터가 삽입된 적이 없다면 새로 Entry를 생성한다. 
        addEntry(hash, key, value, i);
        return null;
    }
```

Java 8에서 성능을 개선하기 위해 변화가 있었습니다.

### Java 8 HashMap에서의 분리 연결법

Java 7까지는 `분리 연결법`에서 충돌이 발생하면 `연결 리스트`를 이용하였습니다. 그런데 이러면 데이터가 많이 쌓였을 때 탐색하는데 시간이 많이 걸린다는 단점이 있기에,

Java 8에서는 일정 개수 이상이 되면 `트리`구조를 이용하는 것으로 발전했습니다. 그러면 O(n)의 탐색시간이 O(logN)으로 빨라질 수 있습니다.

`일정 개수 이상`의 기준은 하나의 버킷에 8개의 `키-값` 쌍이 쌓이면 `리스트 -> 트리`로 변경합니다. 그리고 다시 6개이하가 되면 `트리-> 리스트`의 형태로 바꿉니다.

개수의 차이를 2로 둔 이유는 키-값 쌍이 삽입, 삭제가 연속적으로 일어나게 되었을 때, 계속 리스트->트리, 트리->리스트로 변환하는 과정이 일어나는데 이러면 성능상 좋지 않기 때문에 개수 차이를 2개로 두었습니다.

```java
static final int TREEIFY_THRESHOLD = 8;

static final int UNTREEIFY_THRESHOLD = 6;  
```

Java 8 HashMap에서는 Entry 클래스 대신 Node 클래스를 사용합니다. Node 클래스 자체는 사실상 Java 7의 Entry 클래스와 내용이 같지만, 링크드 리스트 대신 트리를 사용할 수 있도록 하위 클래스인 TreeNode가 있다는 것이 Java 7 HashMap과 다릅니다.

이때 사용하는 트리는 Red-Black Tree인데, Java Collections Framework의 TreeMap과 구현이 거의 같습니다.

### Hash 동적 확장

해시 버킷을 적게 사용한다면 초반에 메모리 사용을 절약할 수 있지만, 충돌을 빈번하게 발생할 수 있다는 단점이 있고, 너무 많이 사용하게 되면 메모리 낭비를 할 수도 있다는 단점이 있습니다.

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    static final float DEFAULT_LOAD_FACTOR = 0.75f;
}
```

그래서 HashMap 클래스를 보면 `기본 용량 = 16`, `로드팩터 : 0.75`를 default로 사용하는 것을 볼 수 있습니다.

여기서 기본용량은 버킷의 수와 같고, 로드팩터는 `(데이터의 개수)/(기본용량)`을 의미합니다. 즉, 로드팩터의 값에 도달하면 버킷의 수를 동적으로 `2배 확장`을 하게 됩니다.

위의 예시로 들면 기본 용량은 16이기 때문에 데이터의 개수가 12개가 차면 버킷의 용량을 16 -> 32로 늘리는 과정이 일어납니다. 이 때 원래 버킷에 있던 것을 새로운 버킷에다 옮기는 과정이 일어나기 때문에 자주 일어난다면 당연히 성능상 좋지 않습니다.

```java
public class HashMap<K,V> extends AbstractMap<K,V>
    implements Map<K,V>, Cloneable, Serializable {

    static final int DEFAULT_INITIAL_CAPACITY = 1 << 4; // aka 16

    static final float DEFAULT_LOAD_FACTOR = 0.75f;

    final float loadFactor;

    public HashMap(int initialCapacity, float loadFactor) {

    }

    public HashMap(int initialCapacity) {
        this(initialCapacity, DEFAULT_LOAD_FACTOR);
    }

    public HashMap() {
        this.loadFactor = DEFAULT_LOAD_FACTOR; // all other fields defaulted
    }
}
```
따라서 생성자로 `기본 용량`, `로드 팩터`를 직접 지정해줄 수 있기 때문에 어느정도 예측 가능한 경우에는 직접 지정해서 사용할 수도 있습니다.

기본 생성자를 사용하면 버킷의 수가 16이기 때문에 계속 2배로 늘어가는 과정이 발생하게 되면 속도가 많이 느려지기 때문에 버킷의 수를 직접 지정하는 것이 성능상 좋습니다.

#### 버킷을 2배로 늘리는데 발생하는 문제점

초기 버킷의 수는 16이고 2배로 늘어나면 32, 64 ... 으로 늘어나게 됩니다. 그러면 2^a 형태가 되는데, 이러면 32비트 영역 중 a개의 비트만 사용하게 된다는 단점이 있습니다. (즉, index = X.hashCode() % M을 계산할 때 X.hashCode()의 하위 a개의 비트만 사용하게 됨)

즉 해시 함수가 32비트 영역을 고르게 사용하도록 만들었다 하더라도 해시 값을 2의 승수로 나누면 해시 충돌이 쉽게 발생할 수 있습니다.

`이래서 보조 해시 함수가 필요합니다.`


### 보조 해시 함수

index = X.hashCode() % M을 계산할 때 사용하는 M 값은 소수일 때 index 값 분포가 가장 균등할 수 있습니다. 그러나 M 값이 소수가 아니기 때문에 별도의 보조 해시 함수를 이용하여 index 값 분포가 가급적 균등할 수 있도록 해야 합니다.

Java 8에서는 위와 같이 Java 7의 보조 해시 함수보다는 훨씬 간단하게 사용하고 있습니다.

```java
static final int hash(Object key) { int h; return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16); }  
```

Java 8 HashMap 보조 해시 함수는 상위 16비트 값을 XOR 연산하는 매우 단순한 형태의 보조 해시 함수를 사용합니다. 이유로는 두 가지가 있는데,

- `첫 번째`는 Java 8에서는 해시 충돌이 많이 발생하면 링크드 리스트 대신 트리를 사용하므로 해시 충돌 시 발생할 수 있는 성능 문제가 완화되었기 때문입니다.
- `두 번째`는 최근의 해시 함수는 균등 분포가 잘 되게 만들어지는 경향이 많아, Java 7까지 사용했던 보조 해시 함수의 효과가 크지 않기 때문입니다. 두 번째 이유가 좀 더 결정적인 원인이 되어 Java 8에서는 보조 해시 함수의 구현을 바꾸었습니다.


### String 객체에 대한 Hash 함수

String의 hash 함수 입니다.
```java
public int hashCode() {  
	int h = hash;
	if (h == 0 && value.length > 0) {
		char val[] = value;

		for (int i = 0; i < value.length; i++) {
			h = 31 * h + val[i];
		}
		hash = h;
	}
	return h;
}
```

여기서 숫자 31을 곱하는 이유는 아래와 같습니다.

- `31이 소수이기 때문입니다.`
- `어떤 수에 31을 곱하면 빠르게 계산할 수 있기 때문입니다. 31N = 32N - N인데, 2^5이니 어떤 수에 대한 32를 곱한 값을 shift 연산으로 쉽게 구현할 수 있습니다. 즉, (N << 5) - N과 같습니다.`


### 참고자료

[Today-I-Learn/Java/Collection/Map/Java HashMap 동작원리.md at master · wjdrbs96/Today-I-Learn · GitHub](https://github.com/wjdrbs96/Today-I-Learn/blob/master/Java/Collection/Map/Java%20HashMap%20%EB%8F%99%EC%9E%91%EC%9B%90%EB%A6%AC.md)