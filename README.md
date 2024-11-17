# advanced-allocator
advanced allocator for cpp.

Almost strickly faster than new/delete. Especially in the case of frequent memory requests

```cpp
#include<bits/stdc++.h>
using namespace std;
const int maxn = 2e5+10;

constexpr int G=24;
#define __lg32(x) (31^__builtin_clz(unsigned(x)))
using uint = unsigned;
int buf[1<<G];
struct node{
	int*p1,*p2;
}blk[2<<G];
struct rang{
	node*p1,*p2;//next: opt out this.
}mbr[G+1];
uint msk;

void init(){
	blk[0]={buf,end(buf)};
	for(int i=0;i<=G;++i){
		mbr[G-i]={blk+((i?2:1)<<i)-1,blk+(2<<i)-1};
	}
	msk=1<<G;
}

inline int* fmalloc(size_t n)noexcept{
	uint let=n==1?1:2<<__lg32(n-1);
	int k=__builtin_ctz(msk&~(let-1));
	node*p=mbr[k].p1++;
	if(mbr[k].p2==mbr[k].p1)msk^=1<<k;
	int*res=p->p1;
	p->p1+=n;
	if(p->p1<p->p2){
		int k=__lg32(p->p2-p->p1);
		msk|=1<<k;
		auto q=--mbr[k].p1;
		*q = *p;
		*p->p1 = uint(q-blk);
	}else*p={};
	return res;
}
inline bool inblk(unsigned i){return i<(sizeof blk/sizeof blk[0]);}
inline void ffree(int*p,size_t n)noexcept{
	int*bg=p,*ed=p+n;
	if(p!=buf){
		if(inblk(*(p-1))) {
			auto&cur=blk[*(p-1)];
			if(cur.p2==p){
				bg=cur.p1;
				int k=__lg32(p-bg);
				cur=*(mbr[k].p1++);
				if(mbr[k].p1==mbr[k].p2)msk^=1<<k;
			}
		}
	}
	if(1){
		if(inblk(*(p+n))) {
			auto&cur=blk[*(p+n)];
			if(cur.p1==p+n){
				ed=cur.p2;
				int k=__lg32(ed-(p+n));
				cur=*(mbr[k].p1++);
				if(mbr[k].p1==mbr[k].p2)msk^=1<<k;
			}
		}
	}
	int k=__lg32(ed-bg);
	msk|=1<<k;
	*--mbr[k].p1={bg,ed};
	*bg=*(ed-1)=uint(mbr[k].p1-blk);
}

template <class T>
class fallocator {
public:
	typedef T           value_type;
	typedef T*          pointer;
	typedef const T*    const_pointer;
	typedef T&          reference;
	typedef const T&    const_reference;
	typedef size_t      size_type;
	typedef ptrdiff_t   difference_type;
	constexpr static size_type rat=(sizeof(T)+sizeof(int)-1)/(sizeof(int));

	pointer allocate(size_type n) {return (pointer)fmalloc(n*rat);}
	void deallocate(pointer p, size_type n) { ffree((int*)p,n*rat); } 
};

namespace test_zhihu{
struct E {int a;E(int aa):a(aa){}};
template<class alloc_t>
void test(alloc_t&&alloc=alloc_t()) {
	cerr<<__PRETTY_FUNCTION__<<": ";
	auto cc=clock();
	static const long long ARRAY_COUNT = 1000L;
	static const long long TEST_COUNT = ARRAY_COUNT * 100000L;
	static E* es[ARRAY_COUNT];
	memset(es,0,sizeof(E*)*ARRAY_COUNT);
	for(long long i=0;i<TEST_COUNT;i++) {
		long long idx=i*123456789L%ARRAY_COUNT;
		E* e = es[idx];
		if(e)e->~E(),alloc.deallocate(e,1);
		es[idx] = alloc.allocate(1);new(es[idx])E((int)i);
	}
	long long n = 0;
	for(long long i=0;i<ARRAY_COUNT;i++) {
		E*e=es[i];
		if(e)n+=e->a;
	}
	cerr<<clock()-cc<<' '<<n<<'\n';
}
}
namespace test_stl_insertend {
	template<class T,int per=1<<18>
	void test(T&&t,unsigned rnd){
		cerr<<__PRETTY_FUNCTION__<<": ";
		auto cur=clock();
		while(rnd--) {
			t.clear();
			for(int i=0;i<per;++i)
				t.insert(t.end(),per-i);
		}
		cerr<<clock()-cur<<"clocks, "<<t.size()<<'\n';
	}
}

signed main() {
	ios::sync_with_stdio(0),cin.tie(0);
	init();//care!

	test_zhihu::test<allocator<test_zhihu::E>>();
	test_zhihu::test<fallocator<test_zhihu::E>>();

	test_stl_insertend::test(vector<int,allocator<int>>(),10000);
	test_stl_insertend::test(vector<int,fallocator<int>>(),10000);

	test_stl_insertend::test(set<int,less<int>,allocator<int>>(),100);
	test_stl_insertend::test(set<int,less<int>,fallocator<int>>(),100);
}
```
