---
layout: post
title: "Moses machine translation performance tuning"
tags: ['Moses', 'performance', 'GCC', 'perf', 'post']
date: 2014-10-24 14:15:30 +0200
comments: true
categories: 
---

Introduction
===

As you maybe know, every year people in SUSE work on projects related to [Hackweek Interstellar](https://hackweek.suse.com/). Idea behind the program is to provide a time slot for people's interests that do not play priority in every day workload. I chose to hack Moses machine translation software. During my studies at university, I met [Aleš Tamchyna](http://ufal.mff.cuni.cz/ales-tamchyna/) who is member of [Institute of Formal and Applied Linguistics](http://ufal.mff.cuni.cz/). I wanted to try a cooperation with someone who has been using an open source project with many instances installed on a quite big computer cluster. Second reason for spending time on the project was to improve my profiling skills, tightly coupled with code generation and assembly language understanding.

Basic analysis
===

As described in the previous section, I chose to utilize [Linux profiling infrastructure](https://perf.wiki.kernel.org/index.php/Tutorial) that presents statistics in user comfortable manner.

``` bash
perf record -g -- ./moses
```

```
    +   8.83%  moses  libtcmalloc_minimal.so.4.1.2  [.] operator new(unsigned long)
    +   3.85%  moses  libc-2.18.so                  [.] __memcmp_sse4_1
    +   3.48%  moses  libtcmalloc_minimal.so.4.1.2  [.] operator delete(void*)
    +   3.39%  moses  moses                         [.] Moses::Hypothesis::~Hypothesis()
    +   2.98%  moses  moses                         [.] Moses::LanguageModelKen<lm::ngram::ProbingModel>::EvaluateWhenApplied(Moses::Hypothesis const&, Moses::FFState const*, Moses::S
    +   2.92%  moses  moses                         [.] Moses::BackwardsEdge::PushSuccessors(unsigned long, unsigned long)
    +   2.85%  moses  moses                         [.] Moses::Hypothesis::RecombineCompare(Moses::Hypothesis const&) const
    +   2.71%  moses  libtcmalloc_minimal.so.4.1.2  [.] tcmalloc::ThreadCache::ReleaseToCentralCache(tcmalloc::ThreadCache::FreeList*, unsigned long, int)
    +   2.70%  moses  moses                         [.] Moses::SearchCubePruning::CheckDistortion(Moses::WordsBitmap const&, Moses::WordsRange const&) const
    +   2.63%  moses  moses                         [.] lm::ngram::detail::GenericModel<lm::ngram::detail::HashedSearch<lm::ngram::BackoffValue>, lm::ngram::ProbingVocabulary>::Resume
    +   2.59%  moses  moses                         [.] void std::__adjust_heap<__gnu_cxx::__normal_iterator<Moses::HypothesisQueueItem**, std::vector<Moses::HypothesisQueueItem*, std
    +   2.58%  moses  moses                         [.] Moses::SquareMatrix::CalcFutureScore(Moses::WordsBitmap const&) const
    +   2.50%  moses  libtcmalloc_minimal.so.4.1.2  [.] tcmalloc::CentralFreeList::FetchFromSpans()
    +   2.15%  moses  perf-32313.map                [.] 0x00007fffff5fcb0f
    +   1.99%  moses  moses                         [.] std::_Rb_tree<int, int, std::_Identity<int>, std::less<int>, std::allocator<int> >::_M_erase(std::_Rb_tree_node<int>*)
    +   1.97%  moses  moses                         [.] Moses::HypothesisScoreOrdererWithDistortion::operator()(Moses::Hypothesis const*, Moses::Hypothesis const*) const
    +   1.69%  moses  libstdc++.so.6.0.21           [.] __dynamic_cast
    +   1.63%  moses  moses                         [.] Moses::Hypothesis::Hypothesis(Moses::Hypothesis const&, Moses::TranslationOption const&)
    +   1.56%  moses  libstdc++.so.6.0.21           [.] std::locale::locale()
    +   1.36%  moses  libstdc++.so.6.0.21           [.] std::locale::~locale()
    +   1.36%  moses  libstdc++.so.6.0.21           [.] std::locale::operator=(std::locale const&)
    +   1.21%  moses  moses                         [.] Moses::Hypothesis::EvaluateWhenApplied(Moses::SquareMatrix const&)
    +   1.15%  moses  moses                         [.] void std::__push_heap<__gnu_cxx::__normal_iterator<Moses::HypothesisQueueItem**, std::vector<Moses::HypothesisQueueItem*, std::
    +   1.10%  moses  moses                         [.] void std::__push_heap<__gnu_cxx::__normal_iterator<Moses::BitmapContainer**, std::vector<Moses::BitmapContainer*, std::allocato
    +   1.10%  moses  moses                         [.] Moses::BackwardsEdge::BackwardsEdge(Moses::BitmapContainer const&, Moses::BitmapContainer&, Moses::TranslationOptionList const&
    +   1.09%  moses  moses                         [.] void std::__adjust_heap<__gnu_cxx::__normal_iterator<Moses::BitmapContainer**, std::vector<Moses::BitmapContainer*, std::alloc
    +   1.09%  moses  moses                         [.] Moses::HypothesisStackCubePruning::Add(Moses::Hypothesis*)
```

Following bunch of observations can be done at first glance:

* memory allocation/deallocation dominates wall time: explanation comes up from fact that Moses creates many Hypothesis and related objects that are quickly evaluated and destroyed; a fast memory allocator would be really beneficial for such kind of application
* \_\_memcmp_sse4\_1: if you append __-g__ for perf, you are given a list of callers for each function; perf presents us that usage of memcmp is uniform distributed for various callers
* Moses::Hypothesis::~Hypothesis looks quite hot with about ~3.5%
* if you dig in functions like Moses::Hypothesis::EvaluateWhenApplied you will see inlined function coming from WordsBitmap.h (more precisely functions GetFirstGapPos and GetNumWordsCovered)
* std::set is chosen as data structure in Moses::BackwardsEdge::PushSuccessors, where m\_seenPosition is utilized just as set container with contains operation; O(log n) is unnecessary complexity for such kind of usage

Optimization improvements
===

Most painfull spots were identified and we tried to reimplement these chunks of codes in faster way. Following three subsections are dedicated to changes in source code.

Reduced dynamic casting
---

Moses executes dynamic casting for every feature function in every Hypothesis class instance:

    const std::vector<FeatureFunction*> &ffs = FeatureFunction::GetFeatureFunctions();
	std::vector<FeatureFunction*>::const_iterator iter;
	for (iter = ffs.begin(); iter != ffs.end(); ++iter) {
	  const FeatureFunction *ff = *iter;

	  const DistortionScoreProducer *model = dynamic_cast<const DistortionScoreProducer*>(ff);
	  if (model) {
	    float weight =staticData.GetAllWeights().GetScoreForProducer(model);
	    m_totalWeightDistortion += weight;
	  }
	}

My modification is presented as Github branch called [distortion-scorer-no-dyncast](https://github.com/marxin/mosesdecoder/tree/distortion-scorer-no-dyncast), where I created a separate static vector s_staticCollDistortion for every DistortionScoreProducer instance registered.

Unordered set
---

Both std C++ (starting from C++11) and boost library contain a data structures that implements unordered set. As Moses community haven't switch to c++11 yes, I decided to use boost::unordered_set as replacement. Patch can be isolated from Github branch named [unordered-set](https://github.com/marxin/mosesdecoder/commits/unordered-set) and code modification comprises just of 2 one-liners.

Bit set representation
---

If you open WordsBitmap.h source file, you can see many bit set operations that return number of one bits in a set, position of first zero(one) bit set; or position of last zero(one) bit. After deep perf report analysis, I identified the most utilized ones:__GetFirstGapPos__ (last zero bit set) and __GetNumWordsCovered__ (well-known as population count).

Original implementation looks straightforward, but slow:
    size_t GetNumWordsCovered() const {
	size_t count = 0;
	for (size_t pos = 0 ; pos < m_size ; pos++) {
	  if (m_bitmap[pos])
	    count++;
	}
	return count;
      }

and

    size_t GetLastGapPos() const {
	for (int pos = (int) m_size - 1 ; pos >= 0 ; pos--) {
	  if (!m_bitmap[pos]) {
	    return pos;
	  }
	}
	// no starting pos
	return NOT_FOUND;
      }

When one has relatively small set, we can use [GCC builtins](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html) presented by compiler: \_\_builtin\_popcount and \_\_builtin_ctz. To fix the problem in more portable manner, I decided to use boost::dynamic_set, which is a data structure tailored to the usage we have. Moreover, __size()__ function (population count) is quite optimal, scanning each byte of a set and accessing precomputed number of bits in such number. For GatLastGapPos, I introduced fast path that can be used for all sets smaller or equal to 64 bits:

    if (m_new_bitmap.size() < UINT_SIZE)
	{
	  unsigned long long v = m_new_bitmap.to_ulong();
	  unsigned index = (unsigned)__builtin_ctzll(~v);
	  return index < m_new_bitmap.size() ? index : 0;
	}

Data container replacement seems quite reasonable. But unfortunatelly, __operator<__ is implemented in opposite way than current implementaion. Suggested code replacement can be applied just in case dynamic_bit implements
opposite [lexical order](http://www.boost.org/doc/libs/1_36_0/libs/dynamic_bitset/dynamic_bitset.html#op-less).

Results
===

For statistics presentation purpose, I decided to pick up two latest official GCC releases: 4.8.1 and 4.9.2. Unfortunatelly, in time of writing this post, GCC 5.0.0 was in stage3 phase and there was a blocker in C++ front-end. We tested 2000 sentences from English to Czech with common preferences on Sandy Bridge i7-4770 CPU @ 3.40GHz with 6 threads enabled in Moses configuration.

<table class="table table-stripped table-bordered table-condensed">
	<thead>
		<tr>
			<th>#</th>
			<th>Configuration</th>
			<th>Description</th>
			<th>Exec. time (s)</th>
			<th>Deviation</th>
			<th>Collation</th>
			<th>Improvement</th>
		</tr>
	</thead>
	<tbody>
		<tr>
			<td>1</td>
			<td>moses-gcc481</td>
			<td>GCC 4.8.1 with tcmalloc</td>
			<td class="tr">17.4154</td>
			<td class="tr">0.17</td>
			<td class="tr">100.00%</td>
			<td class="tr">0.00%</td>
		</tr>
		<tr>
			<td>2</td>
			<td>moses-gcc481-no-tcmalloc</td>
			<td>GCC 4.8.1 w/o tcmalloc</td>
			<td class="tr">18.6173</td>
			<td class="tr">0.15</td>
			<td class="tr">106.90%</td>
			<td class="tr danger">-6.90%</td>
		</tr>
		<tr>
			<td>3</td>
			<td>moses-gcc492</td>
			<td>GCC 4.9.2</td>
			<td class="tr">17.4910</td>
			<td class="tr">0.18</td>
			<td class="tr">100.43%</td>
			<td class="tr danger">-0.43%</td>
		</tr>
		<tr>
			<td>4</td>
			<td>moses-gcc492-lto</td>
			<td>#3 with LTO</td>
			<td class="tr">17.2909</td>
			<td class="tr">0.33</td>
			<td class="tr">99.29%</td>
			<td class="tr success">0.71%</td>
		</tr>
		<tr>
			<td>5</td>
			<td>moses-gcc492-lto-dyn-cast</td>
			<td>#4 w/ reduced casting</td>
			<td class="tr">17.0667</td>
			<td class="tr">0.29</td>
			<td class="tr">98.00%</td>
			<td class="tr success">2.00%</td>
		</tr>
		<tr>
			<td>6</td>
			<td>moses-gcc492-lto-unordered-set</td>
			<td>#5 w/ unordered set</td>
			<td class="tr">16.6062</td>
			<td class="tr">0.23</td>
			<td class="tr">95.35%</td>
			<td class="tr success">4.65%</td>
		</tr>

	</tbody>
</table>

Future work
===

Following performance report illustrates wall time distribution after aforementioned patches were applied.

```
    +   9.90%  moses  libtcmalloc_minimal.so.4.1.2  [.] operator new(unsigned long)
    +   4.68%  moses  moses                         [.] Moses::Hypothesis::EvaluateWhenApplied(Moses::SquareMatrix const&)
    +   3.92%  moses  libtcmalloc_minimal.so.4.1.2  [.] operator delete(void*)
    +   3.92%  moses  moses                         [.] Moses::SearchCubePruning::CreateForwardTodos(Moses::HypothesisStackCubePruning&)
    +   3.63%  moses  moses                         [.] Moses::Hypothesis::~Hypothesis()
    +   3.48%  moses  moses                         [.] void std::__adjust_heap<__gnu_cxx::__normal_iterator<Moses::HypothesisQueueItem**, std::vector<Moses::HypothesisQueueItem*, std::allocator<Moses::HypothesisQueueItem*> > >, long, Moses::HypothesisQueueItem*, __gn
    +   3.37%  moses  moses                         [.] Moses::Hypothesis::RecombineCompare(Moses::Hypothesis const&) const
    +   2.94%  moses  libtcmalloc_minimal.so.4.1.2  [.] tcmalloc::ThreadCache::ReleaseToCentralCache(tcmalloc::ThreadCache::FreeList*, unsigned long, int)
    +   2.88%  moses  moses                         [.] lm::ngram::detail::GenericModel<lm::ngram::detail::HashedSearch<lm::ngram::BackoffValue>, lm::ngram::ProbingVocabulary>::ResumeScore(unsigned int const*, unsigned int const*, unsigned char, unsigned long&, float*
    +   2.65%  moses  moses                         [.] Moses::HypothesisStackCubePruning::~HypothesisStackCubePruning() [clone .lto_priv.2967]
    +   2.65%  moses  moses                         [.] Moses::LanguageModelKen<lm::ngram::ProbingModel>::EvaluateWhenApplied(Moses::Hypothesis const&, Moses::FFState const*, Moses::ScoreComponentCollection*) const
    +   2.54%  moses  libc-2.18.so                  [.] __memcmp_sse4_1
    +   2.36%  moses  libtcmalloc_minimal.so.4.1.2  [.] tcmalloc::CentralFreeList::FetchFromSpans()
    +   2.24%  moses  perf-18464.map                [.] 0x00007fffeab0797d
    +   2.23%  moses  moses                         [.] Moses::BitmapContainer::ProcessBestHypothesis()
    +   1.93%  moses  libstdc++.so.6.0.21           [.] std::locale::locale()
    +   1.71%  moses  libstdc++.so.6.0.21           [.] std::locale::operator=(std::locale const&)
    +   1.68%  moses  libstdc++.so.6.0.21           [.] std::locale::~locale()
    +   1.60%  moses  libstdc++.so.6.0.21           [.] __dynamic_cast
    +   1.46%  moses  moses                         [.] Moses::HypothesisScoreOrdererWithDistortion::operator()(Moses::Hypothesis const*, Moses::Hypothesis const*) const
    +   1.09%  moses  moses                         [.] Moses::BackwardsEdge::CreateHypothesis(Moses::Hypothesis const&, Moses::TranslationOption const&)
```

Possible space for speed improvement:

* Moses::Hypothesis::EvaluateWhenApplied method is dominated by calculation of bit intervals. More precisely, for a given set represented in bits: 010011, we would like identify consecutive zero chunks: <3-4> and <6-6>. I am not familiar with any vector instruction solution which can help
* Moses::SearchCubePruning::CreateForwardTodos heavily executes GetNumWordsCovered, it would be interesting to compare boost implementation with \_\_builtin\_popcount instruction
* Moses::Hypothesis::~Hypothesis is a place where FeatureFunction derives are deallocated; as I visited all derivatives, virtual destructor call looks quite expensive in this situations
* bitset comparison dominates Moses::Hypothesis::RecombineCompare member function, where one can identify if compiler really optimizer fast path

Final thoughts
===

Hacking a performance-intensive open-source project with quite a long history can be really fun. As code base of the project was transformed to C++ with usage of boost library, many abstraction penalties were introduced. If you incorporate fact that the project has just a few active developers, you are given a great acceleration opportunity. I would like to thank Aleš for many source code explanations and I hope this kick-off can be taken by community as hint for further project speed improvements.

Martin Liška (SUSE Labs)
