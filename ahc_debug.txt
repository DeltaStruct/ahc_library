#pragma once
// AHC Human-Only Library v1.5 freeze candidate — single-header distribution.
// AHC_HUMAN_ONLY_SOURCE_URL: REPLACE_WITH_PUBLIC_URL_BEFORE_CONTEST

// ===== ahc/timer.hpp =====
#line 1 "ahc/timer.hpp"
#include <chrono>
#include <algorithm>
#include <cstring>

namespace ahc {

class Timer {
  using Clock = std::chrono::steady_clock;
  Clock::time_point start_ = Clock::now();
  double limit_sec_ = 1e100;
  double safety_sec_ = 0.0;
public:
  Timer() = default;
  explicit Timer(double limit_sec, double safety_sec = 0.0)
      : start_(Clock::now()), limit_sec_(limit_sec), safety_sec_(safety_sec) {}

  void reset(double limit_sec = 1e100, double safety_sec = 0.0) {
    start_ = Clock::now();
    limit_sec_ = limit_sec;
    safety_sec_ = safety_sec;
  }
  [[nodiscard]] double elapsed() const {
    return std::chrono::duration<double>(Clock::now() - start_).count();
  }
  [[nodiscard]] double remaining() const {
    return std::max(0.0, limit_sec_ - safety_sec_ - elapsed());
  }
  [[nodiscard]] bool expired() const { return remaining() <= 0.0; }
  [[nodiscard]] double progress() const {
    const double usable = std::max(1e-12, limit_sec_ - safety_sec_);
    return std::clamp(elapsed() / usable, 0.0, 1.0);
  }

  // Use in hot loops to avoid clock_gettime-equivalent overhead every iteration.
  template<int MASK = 255>
  bool periodic_expired(unsigned long long iteration) const {
    static_assert((MASK & (MASK + 1)) == 0, "MASK must be 2^k-1");
    return ((iteration & MASK) == 0) && expired();
  }
};


// A child budget carved from a parent Timer. Useful for short Beam/LNS/exact phases
// without resetting the global clock.
class TimeSlice {
  const Timer* timer_ = nullptr;
  double begin_ = 0.0;
  double end_ = 0.0;
public:
  TimeSlice() = default;
  TimeSlice(const Timer& timer, double seconds)
      : timer_(&timer), begin_(timer.elapsed()), end_(begin_ + std::max(0.0, seconds)) {}
  static TimeSlice fraction_of_remaining(const Timer& timer, double fraction) {
    return TimeSlice(timer, timer.remaining() * std::clamp(fraction, 0.0, 1.0));
  }
  [[nodiscard]] double elapsed() const { return timer_ ? timer_->elapsed() - begin_ : 0.0; }
  [[nodiscard]] double remaining() const {
    if (!timer_) return 0.0;
    return std::max(0.0, std::min(timer_->remaining(), end_ - timer_->elapsed()));
  }
  [[nodiscard]] bool expired() const { return !timer_ || remaining() <= 0.0; }
  [[nodiscard]] double progress() const {
    if (!timer_ || end_ <= begin_) return 1.0;
    return std::clamp((timer_->elapsed() - begin_) / (end_ - begin_), 0.0, 1.0);
  }
};

class ScopedTimer {
  const Timer& timer_;
  double begin_;
  double* out_;
public:
  ScopedTimer(const Timer& timer, double& out) : timer_(timer), begin_(timer.elapsed()), out_(&out) {}
  ~ScopedTimer() { *out_ += timer_.elapsed() - begin_; }
};

} // namespace ahc

// ===== ahc/rng.hpp =====
#line 1 "ahc/rng.hpp"
#include <cstdint>
#include <algorithm>
#include <iterator>
#include <limits>
#include <array>

namespace ahc {

class Rng {
  uint64_t s_[4];
  static uint64_t rotl(uint64_t x, int k) { return (x << k) | (x >> (64 - k)); }
  static uint64_t splitmix64(uint64_t& x) {
    uint64_t z = (x += 0x9e3779b97f4a7c15ULL);
    z = (z ^ (z >> 30)) * 0xbf58476d1ce4e5b9ULL;
    z = (z ^ (z >> 27)) * 0x94d049bb133111ebULL;
    return z ^ (z >> 31);
  }
public:
  using State = std::array<uint64_t,4>;

  explicit Rng(uint64_t seed = 0x243f6a8885a308d3ULL) { reseed(seed); }
  void reseed(uint64_t seed) {
    for (auto& x : s_) x = splitmix64(seed);
  }
  [[nodiscard]] State state() const { return {s_[0], s_[1], s_[2], s_[3]}; }
  void set_state(const State& st) { for (int i = 0; i < 4; ++i) s_[i] = st[i]; }
  uint64_t u64() {
    const uint64_t result = rotl(s_[0] + s_[3], 23) + s_[0]; // xoshiro256++
    const uint64_t t = s_[1] << 17;
    s_[2] ^= s_[0]; s_[3] ^= s_[1]; s_[1] ^= s_[2]; s_[0] ^= s_[3];
    s_[2] ^= t; s_[3] = rotl(s_[3], 45);
    return result;
  }
  uint32_t u32() { return static_cast<uint32_t>(u64() >> 32); }
  double uniform01() {
    return (u64() >> 11) * (1.0 / 9007199254740992.0);
  }
  int int_range(int l, int r) { // [l, r)
    return l + static_cast<int>(u64() % static_cast<uint64_t>(r - l));
  }
  int uniform_int(int l, int r_inclusive) { return int_range(l, r_inclusive + 1); }
  long long ll_range(long long l, long long r) {
    return l + static_cast<long long>(u64() % static_cast<uint64_t>(r - l));
  }
  long long uniform_ll(long long l, long long r_inclusive) { return ll_range(l, r_inclusive + 1); }
  bool bernoulli(double p) { return uniform01() < p; }
  template<class It>
  void shuffle(It first, It last) {
    auto n = std::distance(first, last);
    for (decltype(n) i = n; i > 1; --i) {
      std::iter_swap(first + (i - 1), first + static_cast<decltype(i)>(u64() % static_cast<uint64_t>(i)));
    }
  }
};

inline uint64_t derive_seed(uint64_t master, uint64_t stream) {
  uint64_t x = master ^ (stream + 0x9e3779b97f4a7c15ULL + (master << 6) + (master >> 2));
  x += 0x9e3779b97f4a7c15ULL;
  x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ULL;
  x = (x ^ (x >> 27)) * 0x94d049bb133111ebULL;
  return x ^ (x >> 31);
}

class SeedStream {
  uint64_t master_;
  uint64_t next_ = 0;
public:
  explicit SeedStream(uint64_t master) : master_(master) {}
  uint64_t next_seed() { return derive_seed(master_, next_++); }
  Rng next_rng() { return Rng(next_seed()); }
  uint64_t index() const { return next_; }
};

} // namespace ahc

// ===== ahc/anneal.hpp =====
#line 1 "ahc/anneal.hpp"
#include <cmath>
#include <algorithm>

namespace ahc {

struct AnnealSchedule {
  double start_temp = 1.0;
  double end_temp = 1e-3;
  double power = 1.0; // >1 keeps temperature higher for longer in log-space.

  [[nodiscard]] double temperature(double progress) const {
    progress = std::clamp(progress, 0.0, 1.0);
    const double q = std::pow(progress, power);
    return start_temp * std::pow(end_temp / start_temp, q);
  }
};

// Minimization. delta = new_score - old_score.
inline bool sa_accept(double delta, double temp, Rng& rng) {
  if (delta <= 0.0) return true;
  if (temp <= 0.0) return false;
  return rng.uniform01() < std::exp(-delta / temp);
}

struct AnnealController {
  AnnealSchedule schedule;
  explicit AnnealController(AnnealSchedule s = {}) : schedule(s) {}
  bool accept(double delta, double progress, Rng& rng) const {
    return sa_accept(delta, schedule.temperature(progress), rng);
  }
};

} // namespace ahc

// ===== ahc/anneal_tune.hpp =====
#line 1 "ahc/anneal_tune.hpp"
#include <vector>
#include <algorithm>
#include <cmath>
#include <limits>

namespace ahc {

// Collect positive (worsening) deltas during a short warm-up and convert their
// quantiles into SA temperatures for desired acceptance probabilities.
class PositiveDeltaSampler {
  std::vector<double> samples_;
public:
  explicit PositiveDeltaSampler(size_t reserve_count = 4096) { samples_.reserve(reserve_count); }
  void clear() { samples_.clear(); }
  void add(double delta) { if (delta > 0.0 && std::isfinite(delta)) samples_.push_back(delta); }
  [[nodiscard]] size_t size() const { return samples_.size(); }

  [[nodiscard]] double quantile(double q) const {
    if (samples_.empty()) return 0.0;
    q = std::clamp(q, 0.0, 1.0);
    std::vector<double> v = samples_;
    size_t k = static_cast<size_t>(q * (v.size() - 1));
    std::nth_element(v.begin(), v.begin() + k, v.end());
    return v[k];
  }

  [[nodiscard]] double temperature_for_acceptance(double accept_probability,
                                                   double delta_quantile = 0.5) const {
    const double d = quantile(delta_quantile);
    if (d <= 0.0) return 0.0;
    const double p = std::clamp(accept_probability, 1e-9, 1.0 - 1e-9);
    return -d / std::log(p);
  }

  [[nodiscard]] AnnealSchedule suggest_schedule(double start_accept = 0.50,
                                                double end_accept = 0.01,
                                                double delta_quantile = 0.5,
                                                double power = 1.0) const {
    double t0 = temperature_for_acceptance(start_accept, delta_quantile);
    double t1 = temperature_for_acceptance(end_accept, delta_quantile);
    if (t0 <= 0.0) t0 = 1.0;
    if (t1 <= 0.0) t1 = std::max(1e-9, t0 * 1e-3);
    if (t1 > t0) std::swap(t0, t1);
    return {t0, t1, power};
  }
};

} // namespace ahc

// ===== ahc/operator_selector.hpp =====
#line 1 "ahc/operator_selector.hpp"
#include <vector>
#include <algorithm>
#include <numeric>
#include <cmath>

namespace ahc {

// Lightweight ALNS-style adaptive roulette.
// reward() may use e.g. +8 new global best, +3 accepted improvement, +1 accepted worsening, 0 rejected.
class AdaptiveOperatorSelector {
  std::vector<double> weight_, reward_sum_;
  std::vector<int> used_;
  double reaction_ = 0.25;
  double floor_ = 0.05;
public:
  explicit AdaptiveOperatorSelector(int n = 0, double reaction = 0.25, double floor = 0.05)
      : weight_(n, 1.0), reward_sum_(n, 0.0), used_(n, 0), reaction_(reaction), floor_(floor) {}

  int size() const { return static_cast<int>(weight_.size()); }
  int choose(Rng& rng) const {
    double sum = 0.0;
    for (double w : weight_) sum += w;
    double x = rng.uniform01() * sum;
    for (int i = 0; i < size(); ++i) {
      x -= weight_[i];
      if (x <= 0.0) return i;
    }
    return size() - 1;
  }
  void reward(int op, double value) {
    reward_sum_[op] += value;
    ++used_[op];
  }
  void update_epoch() {
    for (int i = 0; i < size(); ++i) {
      if (used_[i]) {
        const double target = reward_sum_[i] / used_[i];
        weight_[i] = std::max(floor_, (1.0 - reaction_) * weight_[i] + reaction_ * target);
      }
      reward_sum_[i] = 0.0;
      used_[i] = 0;
    }
  }
  const std::vector<double>& weights() const { return weight_; }
};

} // namespace ahc

// ===== ahc/adaptive_choices.hpp =====
#line 1 "ahc/adaptive_choices.hpp"
#include <vector>
#include <utility>
#include <algorithm>

namespace ahc {

// Adaptive roulette over actual parameter values (destroy size, neighborhood radius, etc.).
template<class T>
class AdaptiveChoices {
  std::vector<T> values_;
  AdaptiveOperatorSelector selector_;
  int epoch_len_ = 256;
  int since_epoch_ = 0;
public:
  struct Pick { int index; const T& value; };

  explicit AdaptiveChoices(std::vector<T> values,
                           int epoch_len = 256,
                           double reaction = 0.25,
                           double floor = 0.05)
      : values_(std::move(values)), selector_(static_cast<int>(values_.size()), reaction, floor),
        epoch_len_(std::max(1, epoch_len)) {}

  Pick choose(Rng& rng) const {
    int i = selector_.choose(rng);
    return {i, values_[i]};
  }
  void reward(int index, double value) {
    selector_.reward(index, value);
    if (++since_epoch_ >= epoch_len_) {
      selector_.update_epoch();
      since_epoch_ = 0;
    }
  }
  void force_epoch_update() { selector_.update_epoch(); since_epoch_ = 0; }
  const std::vector<double>& weights() const { return selector_.weights(); }
  const std::vector<T>& values() const { return values_; }
};

} // namespace ahc

// ===== ahc/lns.hpp =====
#line 1 "ahc/lns.hpp"
#include <algorithm>

namespace ahc {

struct LnsOutcome {
  bool accepted = false;
  bool improved = false;
  bool global_best = false;
  double delta = 0.0; // minimization: new - old
};

inline double default_lns_reward(const LnsOutcome& x) {
  if (x.global_best) return 8.0;
  if (x.accepted && x.improved) return 3.0;
  if (x.accepted) return 1.0;
  return 0.0;
}

// Intentionally small: problem-specific destroy/repair stays outside the library.
// Usage:
//   int op = ctl.choose(rng);
//   auto candidate = destroy_repair(op);
//   bool accept = ctl.accept(candidate.score-current, progress, rng);
//   ... commit/rollback ...
//   ctl.observe(op, {accept, delta<0, new_best, delta});
class LnsController {
  AdaptiveOperatorSelector selector_;
  AnnealController anneal_;
  int epoch_len_ = 256;
  int since_epoch_ = 0;
public:
  LnsController(int operators,
                AnnealSchedule schedule = {1.0, 1e-3, 1.0},
                int epoch_len = 256,
                double reaction = 0.25,
                double floor = 0.05)
      : selector_(operators, reaction, floor), anneal_(schedule), epoch_len_(std::max(1, epoch_len)) {}

  int choose(Rng& rng) const { return selector_.choose(rng); }
  bool accept(double delta, double progress, Rng& rng) const {
    return anneal_.accept(delta, progress, rng);
  }
  void observe(int op, const LnsOutcome& out) {
    selector_.reward(op, default_lns_reward(out));
    if (++since_epoch_ >= epoch_len_) {
      selector_.update_epoch();
      since_epoch_ = 0;
    }
  }
  void force_epoch_update() {
    selector_.update_epoch();
    since_epoch_ = 0;
  }
  const std::vector<double>& weights() const { return selector_.weights(); }
};

} // namespace ahc

// ===== ahc/rollback.hpp =====
#line 1 "ahc/rollback.hpp"
#include <vector>
#include <utility>
#include <cassert>

namespace ahc {

// Rollbackable random-access array. Good for DSU-like/local-search temporary edits.
template<class T>
class RollbackArray {
  std::vector<T> a_;
  std::vector<std::pair<int,T>> history_;
public:
  RollbackArray() = default;
  explicit RollbackArray(int n, const T& init = T()) : a_(n, init) {}
  explicit RollbackArray(std::vector<T> a) : a_(std::move(a)) {}

  const T& operator[](int i) const { return a_[i]; }
  T& raw(int i) { return a_[i]; } // use only when rollback is unnecessary
  int size() const { return static_cast<int>(a_.size()); }
  int snapshot() const { return static_cast<int>(history_.size()); }
  void set(int i, const T& v) {
    if (a_[i] == v) return;
    history_.emplace_back(i, a_[i]);
    a_[i] = v;
  }
  void rollback(int snap) {
    assert(0 <= snap && snap <= static_cast<int>(history_.size()));
    while (static_cast<int>(history_.size()) > snap) {
      auto [i, old] = std::move(history_.back());
      history_.pop_back();
      a_[i] = std::move(old);
    }
  }
  void clear_history() { history_.clear(); }
};

class RollbackGuardBase {
protected:
  bool active_ = true;
public:
  void commit() { active_ = false; }
};

template<class Rollbackable>
class RollbackGuard : public RollbackGuardBase {
  Rollbackable* obj_;
  int snap_;
public:
  explicit RollbackGuard(Rollbackable& obj) : obj_(&obj), snap_(obj.snapshot()) {}
  ~RollbackGuard() { if (active_) obj_->rollback(snap_); }
};

} // namespace ahc

// ===== ahc/rollback_dsu.hpp =====
#line 1 "ahc/rollback_dsu.hpp"
#include <vector>
#include <utility>
#include <cassert>
#include <algorithm>

namespace ahc {

// Rollback DSU: union by size, no path compression.
class RollbackDSU {
  std::vector<int> p_; // negative size for roots
  struct Change { int a, pa, b, pb; };
  std::vector<Change> hist_;
public:
  RollbackDSU() = default;
  explicit RollbackDSU(int n) : p_(n, -1) {}
  void reset(int n) { p_.assign(n, -1); hist_.clear(); }
  [[nodiscard]] int find(int x) const {
    while (p_[x] >= 0) x = p_[x];
    return x;
  }
  [[nodiscard]] bool same(int a, int b) const { return find(a) == find(b); }
  [[nodiscard]] int size(int x) const { return -p_[find(x)]; }
  [[nodiscard]] int snapshot() const { return static_cast<int>(hist_.size()); }

  bool unite(int a, int b) {
    a = find(a); b = find(b);
    if (a == b) return false;
    if (-p_[a] < -p_[b]) std::swap(a, b);
    hist_.push_back({a, p_[a], b, p_[b]});
    p_[a] += p_[b];
    p_[b] = a;
    return true;
  }

  void rollback(int snap) {
    assert(0 <= snap && snap <= static_cast<int>(hist_.size()));
    while (static_cast<int>(hist_.size()) > snap) {
      auto c = hist_.back(); hist_.pop_back();
      p_[c.a] = c.pa;
      p_[c.b] = c.pb;
    }
  }
};

} // namespace ahc

// ===== ahc/arena.hpp =====
#line 1 "ahc/arena.hpp"
#include <vector>
#include <utility>
#include <cassert>
#include <cstddef>

namespace ahc {

// Reusable object pool backed by one vector allocation.
// reset() destroys objects but keeps capacity, so repeated Beam/LNS phases do not churn the heap.
template<class T>
class ObjectArena {
  std::vector<T> data_;
public:
  ObjectArena() = default;
  explicit ObjectArena(size_t reserve_count) { data_.reserve(reserve_count); }

  void reserve(size_t n) { data_.reserve(n); }
  void reset() { data_.clear(); }
  [[nodiscard]] size_t size() const { return data_.size(); }
  [[nodiscard]] size_t capacity() const { return data_.capacity(); }

  template<class... Args>
  int emplace(Args&&... args) {
    data_.emplace_back(std::forward<Args>(args)...);
    return static_cast<int>(data_.size()) - 1;
  }
  int push(const T& value) {
    data_.push_back(value);
    return static_cast<int>(data_.size()) - 1;
  }
  int push(T&& value) {
    data_.push_back(std::move(value));
    return static_cast<int>(data_.size()) - 1;
  }

  T& operator[](int i) { assert(0 <= i && static_cast<size_t>(i) < data_.size()); return data_[i]; }
  const T& operator[](int i) const { assert(0 <= i && static_cast<size_t>(i) < data_.size()); return data_[i]; }

  auto begin() { return data_.begin(); }
  auto end() { return data_.end(); }
  auto begin() const { return data_.begin(); }
  auto end() const { return data_.end(); }
};

// Small fixed-capacity candidate buffer. Great for "keep best few per parent"
// without allocating vectors inside the hot loop. Lower score is better.
template<class T, int CAP, class ScoreFn>
class SmallTopK {
  static_assert(CAP > 0);
  T a_[CAP];
  int n_ = 0;
  ScoreFn score_;
public:
  explicit SmallTopK(ScoreFn score = {}) : score_(score) {}
  void clear() { n_ = 0; }
  int size() const { return n_; }
  const T& operator[](int i) const { return a_[i]; }

  void push(const T& x) {
    if (n_ < CAP) {
      a_[n_++] = x;
      for (int i = n_ - 1; i > 0 && score_(a_[i]) < score_(a_[i-1]); --i)
        std::swap(a_[i], a_[i-1]);
      return;
    }
    if (!(score_(x) < score_(a_[CAP-1]))) return;
    a_[CAP-1] = x;
    for (int i = CAP - 1; i > 0 && score_(a_[i]) < score_(a_[i-1]); --i)
      std::swap(a_[i], a_[i-1]);
  }
};

} // namespace ahc

// ===== ahc/static_vector.hpp =====
#line 1 "ahc/static_vector.hpp"
#include <array>
#include <cassert>
#include <utility>
#include <algorithm>

namespace ahc {

// Tiny allocation-free vector for small fixed-capacity state components.
template<class T, int CAP>
class StaticVector {
  static_assert(CAP >= 0);
  std::array<T, CAP> a_{};
  int n_ = 0;
public:
  using value_type = T;
  [[nodiscard]] int size() const { return n_; }
  [[nodiscard]] constexpr int capacity() const { return CAP; }
  [[nodiscard]] bool empty() const { return n_ == 0; }
  void clear() { n_ = 0; }
  T& operator[](int i) { assert(0 <= i && i < n_); return a_[i]; }
  const T& operator[](int i) const { assert(0 <= i && i < n_); return a_[i]; }
  T& back() { assert(n_); return a_[n_ - 1]; }
  const T& back() const { assert(n_); return a_[n_ - 1]; }
  void push_back(const T& x) { assert(n_ < CAP); a_[n_++] = x; }
  void push_back(T&& x) { assert(n_ < CAP); a_[n_++] = std::move(x); }
  template<class... Args> T& emplace_back(Args&&... args) {
    assert(n_ < CAP);
    a_[n_] = T(std::forward<Args>(args)...);
    return a_[n_++];
  }
  void pop_back() { assert(n_); --n_; }
  void erase_swap(int i) {
    assert(0 <= i && i < n_);
    if (i + 1 != n_) a_[i] = std::move(a_[n_ - 1]);
    --n_;
  }
  T* begin() { return a_.data(); }
  T* end() { return a_.data() + n_; }
  const T* begin() const { return a_.data(); }
  const T* end() const { return a_.data() + n_; }
};

} // namespace ahc

// ===== ahc/history_arena.hpp =====
#line 1 "ahc/history_arena.hpp"
#include <vector>
#include <algorithm>

namespace ahc {

template<class Action>
class HistoryArena {
public:
  struct Node { int parent; Action action; };
private:
  std::vector<Node> nodes_;
public:
  explicit HistoryArena(size_t reserve_nodes = 0) {
    if (reserve_nodes) nodes_.reserve(reserve_nodes);
  }
  void clear() { nodes_.clear(); }
  int add(int parent, const Action& action) {
    nodes_.push_back({parent, action});
    return static_cast<int>(nodes_.size()) - 1;
  }
  int add(int parent, Action&& action) {
    nodes_.push_back({parent, std::move(action)});
    return static_cast<int>(nodes_.size()) - 1;
  }
  std::vector<Action> restore(int node) const {
    std::vector<Action> out;
    while (node >= 0) {
      out.push_back(nodes_[node].action);
      node = nodes_[node].parent;
    }
    std::reverse(out.begin(), out.end());
    return out;
  }
  size_t size() const { return nodes_.size(); }
};

} // namespace ahc

// ===== ahc/zobrist.hpp =====
#line 1 "ahc/zobrist.hpp"
#include <vector>
#include <cstdint>

namespace ahc {

class ZobristTable {
  int rows_ = 0, cols_ = 0;
  std::vector<uint64_t> table_;
public:
  ZobristTable() = default;
  ZobristTable(int rows, int cols, uint64_t seed = 1) { reset(rows, cols, seed); }
  void reset(int rows, int cols, uint64_t seed = 1) {
    rows_ = rows; cols_ = cols;
    table_.resize(static_cast<size_t>(rows) * cols);
    Rng rng(seed);
    for (auto& x : table_) x = rng.u64();
  }
  uint64_t operator()(int r, int c) const { return table_[static_cast<size_t>(r) * cols_ + c]; }
  uint64_t update(uint64_t h, int r, int old_c, int new_c) const {
    if (old_c >= 0) h ^= (*this)(r, old_c);
    if (new_c >= 0) h ^= (*this)(r, new_c);
    return h;
  }
};

} // namespace ahc

// ===== ahc/flat_hash.hpp =====
#line 1 "ahc/flat_hash.hpp"
#include <vector>
#include <algorithm>
#include <cstdint>
#include <cassert>
#include <limits>
#include <utility>

namespace ahc {

inline uint64_t mix64(uint64_t x) {
  x += 0x9e3779b97f4a7c15ULL;
  x = (x ^ (x >> 30)) * 0xbf58476d1ce4e5b9ULL;
  x = (x ^ (x >> 27)) * 0x94d049bb133111ebULL;
  return x ^ (x >> 31);
}

// Open-addressing hash set specialized for uint64_t keys.
// clear() is O(1) via generation stamps. Reserve enough capacity up front;
// intended for Beam dedup / visited tables in hot loops.
class FlatHashSet64 {
  std::vector<uint64_t> keys_;
  std::vector<uint32_t> stamp_;
  uint32_t generation_ = 1;
  size_t mask_ = 0;
  size_t size_ = 0;

  static size_t ceil_pow2(size_t x) {
    size_t p = 1;
    while (p < x) p <<= 1;
    return p;
  }

public:
  FlatHashSet64() { reset_capacity(16); }
  explicit FlatHashSet64(size_t expected_items) { reserve(expected_items); }

  void reserve(size_t expected_items) {
    // Keep load <= 0.5. This is intentionally simple and predictable.
    reset_capacity(ceil_pow2(std::max<size_t>(16, expected_items * 2 + 1)));
  }

  void reset_capacity(size_t cap_pow2) {
    assert(cap_pow2 && (cap_pow2 & (cap_pow2 - 1)) == 0);
    keys_.assign(cap_pow2, 0);
    stamp_.assign(cap_pow2, 0);
    mask_ = cap_pow2 - 1;
    generation_ = 1;
    size_ = 0;
  }

  void clear() {
    size_ = 0;
    ++generation_;
    if (generation_ == 0) {
      std::fill(stamp_.begin(), stamp_.end(), 0);
      generation_ = 1;
    }
  }

  [[nodiscard]] size_t size() const { return size_; }
  [[nodiscard]] size_t capacity() const { return keys_.size(); }

  bool insert(uint64_t key) {
    // Do not silently loop forever if the table was undersized.
    assert((size_ + 1) * 10 <= keys_.size() * 7 && "FlatHashSet64 capacity too small");
    size_t p = static_cast<size_t>(mix64(key)) & mask_;
    while (stamp_[p] == generation_) {
      if (keys_[p] == key) return false;
      p = (p + 1) & mask_;
    }
    stamp_[p] = generation_;
    keys_[p] = key;
    ++size_;
    return true;
  }

  [[nodiscard]] bool contains(uint64_t key) const {
    size_t p = static_cast<size_t>(mix64(key)) & mask_;
    while (stamp_[p] == generation_) {
      if (keys_[p] == key) return true;
      p = (p + 1) & mask_;
    }
    return false;
  }
};

// Same idea, but stores a small integer counter for each uint64_t key.
// Useful for Beam per-bucket caps without unordered_map.
class FlatHashCounter64 {
  std::vector<uint64_t> keys_;
  std::vector<int> values_;
  std::vector<uint32_t> stamp_;
  uint32_t generation_ = 1;
  size_t mask_ = 0;
  size_t size_ = 0;

  static size_t ceil_pow2(size_t x) {
    size_t p = 1;
    while (p < x) p <<= 1;
    return p;
  }

public:
  FlatHashCounter64() { reserve(16); }
  explicit FlatHashCounter64(size_t expected_items) { reserve(expected_items); }

  void reserve(size_t expected_items) {
    const size_t cap = ceil_pow2(std::max<size_t>(16, expected_items * 2 + 1));
    keys_.assign(cap, 0);
    values_.assign(cap, 0);
    stamp_.assign(cap, 0);
    mask_ = cap - 1;
    generation_ = 1;
    size_ = 0;
  }

  void clear() {
    size_ = 0;
    ++generation_;
    if (generation_ == 0) {
      std::fill(stamp_.begin(), stamp_.end(), 0);
      generation_ = 1;
    }
  }
  [[nodiscard]] size_t size() const { return size_; }
  [[nodiscard]] size_t capacity() const { return keys_.size(); }

  int& get(uint64_t key) {
    assert((size_ + 1) * 10 <= keys_.size() * 7 && "FlatHashCounter64 capacity too small");
    size_t p = static_cast<size_t>(mix64(key)) & mask_;
    while (stamp_[p] == generation_) {
      if (keys_[p] == key) return values_[p];
      p = (p + 1) & mask_;
    }
    stamp_[p] = generation_;
    keys_[p] = key;
    values_[p] = 0;
    ++size_;
    return values_[p];
  }
};


// Generation-stamped uint64_t -> T map. Useful as a transposition table storing
// best known evaluation/depth for the current Beam layer or DFS iteration.
template<class T>
class FlatHashMap64 {
  std::vector<uint64_t> keys_;
  std::vector<T> values_;
  std::vector<uint32_t> stamp_;
  uint32_t generation_ = 1;
  size_t mask_ = 0;
  size_t size_ = 0;

  static size_t ceil_pow2(size_t x) {
    size_t p = 1;
    while (p < x) p <<= 1;
    return p;
  }
public:
  FlatHashMap64() { reserve(16); }
  explicit FlatHashMap64(size_t expected_items) { reserve(expected_items); }

  void reserve(size_t expected_items) {
    const size_t cap = ceil_pow2(std::max<size_t>(16, expected_items * 2 + 1));
    keys_.assign(cap, 0);
    values_.resize(cap);
    stamp_.assign(cap, 0);
    mask_ = cap - 1;
    generation_ = 1;
    size_ = 0;
  }
  void clear() {
    size_ = 0;
    ++generation_;
    if (generation_ == 0) {
      std::fill(stamp_.begin(), stamp_.end(), 0);
      generation_ = 1;
    }
  }
  [[nodiscard]] size_t size() const { return size_; }
  [[nodiscard]] size_t capacity() const { return keys_.size(); }

  T* find(uint64_t key) {
    size_t p = static_cast<size_t>(mix64(key)) & mask_;
    while (stamp_[p] == generation_) {
      if (keys_[p] == key) return &values_[p];
      p = (p + 1) & mask_;
    }
    return nullptr;
  }
  const T* find(uint64_t key) const {
    size_t p = static_cast<size_t>(mix64(key)) & mask_;
    while (stamp_[p] == generation_) {
      if (keys_[p] == key) return &values_[p];
      p = (p + 1) & mask_;
    }
    return nullptr;
  }
  T& get_or_insert(uint64_t key, const T& init = T()) {
    assert((size_ + 1) * 10 <= keys_.size() * 7 && "FlatHashMap64 capacity too small");
    size_t p = static_cast<size_t>(mix64(key)) & mask_;
    while (stamp_[p] == generation_) {
      if (keys_[p] == key) return values_[p];
      p = (p + 1) & mask_;
    }
    stamp_[p] = generation_;
    keys_[p] = key;
    values_[p] = init;
    ++size_;
    return values_[p];
  }

  template<class Better>
  bool update_if_better(uint64_t key, const T& value, Better better) {
    T* old = find(key);
    if (old) {
      if (better(value, *old)) { *old = value; return true; }
      return false;
    }
    get_or_insert(key, value);
    return true;
  }
};

} // namespace ahc

// ===== ahc/stamped_array.hpp =====
#line 1 "ahc/stamped_array.hpp"
#include <vector>
#include <cstdint>
#include <algorithm>
#include <cassert>

namespace ahc {

// O(1) logical clear for arrays indexed by a dense integer id.
template<class T>
class StampedArray {
  std::vector<T> value_;
  std::vector<uint32_t> stamp_;
  uint32_t generation_ = 1;
public:
  StampedArray() = default;
  explicit StampedArray(int n) { reset(n); }
  void reset(int n) {
    value_.resize(n);
    stamp_.assign(n, 0);
    generation_ = 1;
  }
  void clear() {
    ++generation_;
    if (generation_ == 0) {
      std::fill(stamp_.begin(), stamp_.end(), 0);
      generation_ = 1;
    }
  }
  [[nodiscard]] int size() const { return static_cast<int>(value_.size()); }
  [[nodiscard]] bool contains(int i) const {
    assert(0 <= i && i < size());
    return stamp_[i] == generation_;
  }
  void set(int i, const T& v) {
    assert(0 <= i && i < size());
    stamp_[i] = generation_;
    value_[i] = v;
  }
  T& get_or_init(int i, const T& init = T()) {
    assert(0 <= i && i < size());
    if (stamp_[i] != generation_) {
      stamp_[i] = generation_;
      value_[i] = init;
    }
    return value_[i];
  }
  const T& get(int i) const {
    assert(contains(i));
    return value_[i];
  }
};

} // namespace ahc

// ===== ahc/stats.hpp =====
#line 1 "ahc/stats.hpp"
#include <cmath>
#include <limits>
#include <string>
#include <sstream>
#include <iomanip>

namespace ahc {

struct OnlineStats {
  long long n = 0;
  double mean = 0.0, m2 = 0.0;
  double min = std::numeric_limits<double>::infinity();
  double max = -std::numeric_limits<double>::infinity();
  void add(double x) {
    ++n;
    const double d = x - mean;
    mean += d / n;
    m2 += d * (x - mean);
    if (x < min) min = x;
    if (x > max) max = x;
  }
  double variance() const { return n > 1 ? m2 / (n - 1) : 0.0; }
  double stdev() const { return std::sqrt(variance()); }
};

struct MoveStats {
  long long proposed = 0, accepted = 0, improved = 0, best = 0;
  double total_delta = 0.0;
  void proposal() { ++proposed; }
  void acceptance(double delta, bool global_best = false) {
    ++accepted;
    total_delta += delta;
    if (delta < 0) ++improved;
    if (global_best) ++best;
  }
  double accept_rate() const { return proposed ? double(accepted) / proposed : 0.0; }
  double improve_rate() const { return proposed ? double(improved) / proposed : 0.0; }
  double best_rate() const { return proposed ? double(best) / proposed : 0.0; }
  double mean_accepted_delta() const { return accepted ? total_delta / accepted : 0.0; }
};

} // namespace ahc

// ===== ahc/delta_debug.hpp =====
#line 1 "ahc/delta_debug.hpp"
#include <cassert>
#include <cmath>
#include <iostream>
#include <type_traits>

namespace ahc {

// Periodically verifies incremental score against a full recomputation.
// Enable aggressively during development, then increase period / disable for submission.
class DeltaChecker {
  unsigned long long period_ = 1;
  unsigned long long calls_ = 0;
  double eps_ = 1e-9;
  bool enabled_ = true;
public:
  explicit DeltaChecker(unsigned long long period = 1, double eps = 1e-9)
      : period_(period ? period : 1), eps_(eps) {}

  void enable(bool x = true) { enabled_ = x; }

  template<class Score, class FullScoreFn>
  void check(const Score& incremental_score, FullScoreFn&& full_score_fn,
             const char* label = "delta") {
    if (!enabled_) return;
    if ((calls_++ % period_) != 0) return;
    const auto full = full_score_fn();
    bool ok;
    if constexpr (std::is_floating_point_v<Score> || std::is_floating_point_v<decltype(full)>) {
      const double a = static_cast<double>(incremental_score);
      const double b = static_cast<double>(full);
      ok = std::abs(a - b) <= eps_ * (1.0 + std::max(std::abs(a), std::abs(b)));
    } else {
      ok = (incremental_score == full);
    }
    if (!ok) {
      std::cerr << "[DeltaChecker] " << label
                << " incremental=" << incremental_score
                << " full=" << full << '\n';
      assert(ok && "incremental score mismatch");
    }
  }

  template<class Score>
  void check_transition(const Score& old_score, const Score& delta,
                        const Score& new_score, const char* label = "transition") {
    const auto predicted = old_score + delta;
    bool ok;
    if constexpr (std::is_floating_point_v<Score>) {
      const double a = static_cast<double>(predicted), b = static_cast<double>(new_score);
      ok = std::abs(a - b) <= eps_ * (1.0 + std::max(std::abs(a), std::abs(b)));
    } else {
      ok = (predicted == new_score);
    }
    if (!ok) {
      std::cerr << "[DeltaChecker] " << label
                << " old=" << old_score << " delta=" << delta
                << " predicted=" << predicted << " actual=" << new_score << '\n';
      assert(ok && "delta transition mismatch");
    }
  }
};

} // namespace ahc

// ===== ahc/csv_logger.hpp =====
#line 1 "ahc/csv_logger.hpp"
#include <fstream>
#include <sstream>
#include <string>
#include <string_view>
#include <utility>

namespace ahc {

inline std::string csv_escape(std::string_view s) {
  bool quote = false;
  for (char c : s) if (c == ',' || c == '"' || c == '\n' || c == '\r') { quote = true; break; }
  if (!quote) return std::string(s);
  std::string out = "\"";
  for (char c : s) {
    if (c == '"') out += "\"\"";
    else out += c;
  }
  out += '"';
  return out;
}

class CsvLogger {
  std::ofstream out_;

  template<class T>
  static std::string cell(const T& x) {
    std::ostringstream ss;
    ss << x;
    return csv_escape(ss.str());
  }
  static std::string cell(const std::string& s) { return csv_escape(s); }
  static std::string cell(std::string_view s) { return csv_escape(s); }
  static std::string cell(const char* s) { return csv_escape(s ? std::string_view(s) : std::string_view()); }

public:
  CsvLogger() = default;
  explicit CsvLogger(const std::string& path, bool append = false) { open(path, append); }

  bool open(const std::string& path, bool append = false) {
    out_.open(path, append ? (std::ios::out | std::ios::app) : std::ios::out);
    return out_.good();
  }
  bool good() const { return out_.good(); }
  void flush() { out_.flush(); }

  template<class First, class... Rest>
  void row(const First& first, const Rest&... rest) {
    out_ << cell(first);
    ((out_ << ',' << cell(rest)), ...);
    out_ << '\n';
  }
};

} // namespace ahc

// ===== ahc/benchmark.hpp =====
#line 1 "ahc/benchmark.hpp"
#include <vector>
#include <algorithm>
#include <numeric>
#include <cmath>
#include <sstream>
#include <string>
#include <iomanip>
#include <cstdint>

namespace ahc {

inline double quantile(std::vector<double> v, double q) {
  if (v.empty()) return 0.0;
  q = std::clamp(q, 0.0, 1.0);
  const size_t k = static_cast<size_t>(q * (v.size() - 1));
  std::nth_element(v.begin(), v.begin() + k, v.end());
  return v[k];
}
inline double median(std::vector<double> v) { return quantile(std::move(v), 0.5); }

// Stable train/holdout split from the seed itself. This prevents manually moving
// troublesome cases between sets while tuning.
inline bool is_holdout(uint64_t seed, int numerator = 1, int denominator = 5,
                       uint64_t salt = 0x5a17d3e4c9b8126fULL) {
  if (denominator <= 0) return false;
  numerator = std::clamp(numerator, 0, denominator);
  return static_cast<int>(derive_seed(seed, salt) % static_cast<uint64_t>(denominator)) < numerator;
}

struct PairedSummary {
  int n = 0, wins = 0, ties = 0, losses = 0; // lower score is better: challenger vs baseline
  double baseline_mean = 0.0;
  double challenger_mean = 0.0;
  double mean_delta = 0.0;       // challenger - baseline
  double median_delta = 0.0;
  double p90_delta = 0.0;        // regression-tail indicator
  double p95_delta = 0.0;
  double best_delta = 0.0;
  double worst_delta = 0.0;
  double geometric_mean_ratio = 1.0; // challenger / baseline for positive scores

  [[nodiscard]] double win_rate_excluding_ties() const {
    return wins + losses ? double(wins) / (wins + losses) : 0.5;
  }

  std::string str(int precision = 3) const {
    std::ostringstream ss;
    ss << std::fixed << std::setprecision(precision)
       << "n=" << n << " W/T/L=" << wins << '/' << ties << '/' << losses
       << " baseline_mean=" << baseline_mean
       << " challenger_mean=" << challenger_mean
       << " mean_delta=" << mean_delta
       << " median_delta=" << median_delta
       << " p90_delta=" << p90_delta
       << " p95_delta=" << p95_delta
       << " worst_delta=" << worst_delta
       << " geo_ratio=" << geometric_mean_ratio;
    return ss.str();
  }
};

inline PairedSummary summarize_paired(const std::vector<double>& baseline,
                                      const std::vector<double>& challenger,
                                      double tie_eps = 1e-12) {
  const int n = static_cast<int>(std::min(baseline.size(), challenger.size()));
  PairedSummary s; s.n = n;
  if (!n) return s;
  std::vector<double> delta; delta.reserve(n);
  double log_ratio_sum = 0.0;
  int ratio_n = 0;
  for (int i = 0; i < n; ++i) {
    const double d = challenger[i] - baseline[i];
    delta.push_back(d);
    s.baseline_mean += baseline[i];
    s.challenger_mean += challenger[i];
    if (d < -tie_eps) ++s.wins;
    else if (d > tie_eps) ++s.losses;
    else ++s.ties;
    if (baseline[i] > 0.0 && challenger[i] > 0.0) {
      log_ratio_sum += std::log(challenger[i] / baseline[i]);
      ++ratio_n;
    }
  }
  s.baseline_mean /= n;
  s.challenger_mean /= n;
  s.mean_delta = std::accumulate(delta.begin(), delta.end(), 0.0) / n;
  s.median_delta = quantile(delta, 0.5);
  s.p90_delta = quantile(delta, 0.90);
  s.p95_delta = quantile(delta, 0.95);
  s.best_delta = *std::min_element(delta.begin(), delta.end());
  s.worst_delta = *std::max_element(delta.begin(), delta.end());
  s.geometric_mean_ratio = ratio_n ? std::exp(log_ratio_sum / ratio_n) : 1.0;
  return s;
}

} // namespace ahc

// ===== ahc/best_keeper.hpp =====
#line 1 "ahc/best_keeper.hpp"
#include <functional>
#include <optional>
#include <utility>

namespace ahc {

// Stores the current best state. Copying only happens on an actual improvement.
// Better(a,b)==true means score a is better than score b.
template<class State, class Score, class Better = std::less<Score>>
class BestKeeper {
  std::optional<State> state_;
  std::optional<Score> score_;
  Better better_{};
public:
  BestKeeper() = default;
  explicit BestKeeper(Better better) : better_(std::move(better)) {}
  BestKeeper(const State& state, const Score& score, Better better = {})
      : state_(state), score_(score), better_(std::move(better)) {}

  [[nodiscard]] bool has_value() const { return state_.has_value(); }
  [[nodiscard]] const State& state() const { return *state_; }
  [[nodiscard]] State& state() { return *state_; }
  [[nodiscard]] const Score& score() const { return *score_; }

  bool update(const State& state, const Score& score) {
    if (!score_ || better_(score, *score_)) {
      state_ = state;
      score_ = score;
      return true;
    }
    return false;
  }

  bool update(State&& state, const Score& score) {
    if (!score_ || better_(score, *score_)) {
      state_ = std::move(state);
      score_ = score;
      return true;
    }
    return false;
  }
};

} // namespace ahc

// ===== ahc/elite_pool.hpp =====
#line 1 "ahc/elite_pool.hpp"
#include <vector>
#include <algorithm>
#include <cassert>
#include <cstdint>
#include <functional>

namespace ahc {

// Quality + diversity pool for restart states. Diversity is supplied as a coarse bucket.
// At most per_bucket entries are retained from the same bucket.
template<class State, class Score, class Better = std::less<Score>>
class ElitePool {
public:
  struct Entry { State state; Score score; uint64_t bucket; };
private:
  int capacity_ = 8;
  int per_bucket_ = 1;
  Better better_{};
  std::vector<Entry> a_;

  int worst_index() const {
    int w = 0;
    for (int i = 1; i < static_cast<int>(a_.size()); ++i)
      if (better_(a_[w].score, a_[i].score)) w = i;
    return w;
  }
public:
  explicit ElitePool(int capacity = 8, int per_bucket = 1, Better better = {})
      : capacity_(std::max(1, capacity)), per_bucket_(std::max(1, per_bucket)), better_(std::move(better)) {
    a_.reserve(capacity_);
  }

  [[nodiscard]] int size() const { return static_cast<int>(a_.size()); }
  [[nodiscard]] bool empty() const { return a_.empty(); }
  [[nodiscard]] const std::vector<Entry>& entries() const { return a_; }

  bool insert(const State& state, const Score& score, uint64_t bucket) {
    int same_count = 0, worst_same = -1;
    for (int i = 0; i < size(); ++i) if (a_[i].bucket == bucket) {
      ++same_count;
      if (worst_same < 0 || better_(a_[worst_same].score, a_[i].score)) worst_same = i;
    }
    if (same_count >= per_bucket_) {
      if (worst_same >= 0 && better_(score, a_[worst_same].score)) {
        a_[worst_same] = {state, score, bucket};
        return true;
      }
      return false;
    }
    if (size() < capacity_) {
      a_.push_back({state, score, bucket});
      return true;
    }
    int w = worst_index();
    if (better_(score, a_[w].score)) {
      a_[w] = {state, score, bucket};
      return true;
    }
    return false;
  }

  const Entry& best() const {
    assert(!a_.empty());
    int b = 0;
    for (int i = 1; i < size(); ++i) if (better_(a_[i].score, a_[b].score)) b = i;
    return a_[b];
  }

  const Entry& sample_uniform(Rng& rng) const {
    assert(!a_.empty());
    return a_[rng.int_range(0, size())];
  }

  // bias=0 => uniform. Larger bias favors better ranks while still allowing diverse restarts.
  const Entry& sample_rank_biased(Rng& rng, double bias = 2.0) const {
    assert(!a_.empty());
    std::vector<int> ord(size());
    for (int i = 0; i < size(); ++i) ord[i] = i;
    std::sort(ord.begin(), ord.end(), [&](int x, int y){ return better_(a_[x].score, a_[y].score); });
    if (bias <= 0.0) return a_[ord[rng.int_range(0, size())]];
    double u = rng.uniform01();
    int rank = std::min(size() - 1, static_cast<int>(std::pow(u, bias) * size()));
    return a_[ord[rank]];
  }
};

} // namespace ahc

// ===== ahc/phase.hpp =====
#line 1 "ahc/phase.hpp"
#include <vector>
#include <algorithm>
#include <cassert>

namespace ahc {

// Normalized time phases. Example boundaries {0.08, 0.80, 0.96, 1.0}
// create phases 0..3. local_progress() maps the current phase to [0,1].
class PhaseSchedule {
  std::vector<double> end_;
public:
  PhaseSchedule() : end_{1.0} {}
  explicit PhaseSchedule(std::vector<double> ends) : end_(std::move(ends)) {
    assert(!end_.empty());
    for (double& x : end_) x = std::clamp(x, 0.0, 1.0);
    assert(std::is_sorted(end_.begin(), end_.end()));
    if (end_.back() < 1.0) end_.push_back(1.0);
  }

  [[nodiscard]] int count() const { return static_cast<int>(end_.size()); }
  [[nodiscard]] int phase(double progress) const {
    progress = std::clamp(progress, 0.0, 1.0);
    return static_cast<int>(std::lower_bound(end_.begin(), end_.end(), progress) - end_.begin());
  }
  [[nodiscard]] double begin(int phase) const { return phase == 0 ? 0.0 : end_[phase - 1]; }
  [[nodiscard]] double end(int phase) const { return end_[phase]; }
  [[nodiscard]] double local_progress(double progress) const {
    const int p = phase(progress);
    const double l = begin(p), r = end(p);
    if (r <= l + 1e-15) return 1.0;
    return std::clamp((progress - l) / (r - l), 0.0, 1.0);
  }
};

} // namespace ahc

// ===== ahc/stagnation.hpp =====
#line 1 "ahc/stagnation.hpp"
#include <limits>
#include <algorithm>

namespace ahc {

// Tracks how long it has been since the last global improvement.
class StagnationTracker {
  double best_ = std::numeric_limits<double>::infinity();
  unsigned long long last_iter_ = 0;
  double last_time_ = 0.0;
  bool initialized_ = false;
public:
  bool update(double score, unsigned long long iter, double elapsed) {
    if (!initialized_ || score < best_) {
      best_ = score;
      last_iter_ = iter;
      last_time_ = elapsed;
      initialized_ = true;
      return true;
    }
    return false;
  }
  [[nodiscard]] double best() const { return best_; }
  [[nodiscard]] unsigned long long stagnant_iters(unsigned long long iter) const {
    return initialized_ ? iter - last_iter_ : 0;
  }
  [[nodiscard]] double stagnant_seconds(double elapsed) const {
    return initialized_ ? std::max(0.0, elapsed - last_time_) : 0.0;
  }
  [[nodiscard]] bool stalled(unsigned long long iter, double elapsed,
                             unsigned long long iter_limit, double sec_limit) const {
    return (iter_limit && stagnant_iters(iter) >= iter_limit) ||
           (sec_limit > 0.0 && stagnant_seconds(elapsed) >= sec_limit);
  }
};

} // namespace ahc

// ===== ahc/profiler.hpp =====
#line 1 "ahc/profiler.hpp"
#include <chrono>
#include <string>
#include <vector>
#include <sstream>
#include <iomanip>
#include <algorithm>

namespace ahc {

class Profiler {
  using Clock = std::chrono::steady_clock;
public:
  struct Entry { std::string name; double seconds = 0.0; unsigned long long calls = 0; };
private:
  std::vector<Entry> e_;
public:
  int add(std::string name) {
    e_.push_back({std::move(name), 0.0, 0});
    return static_cast<int>(e_.size()) - 1;
  }
  void record(int id, double sec) { e_[id].seconds += sec; ++e_[id].calls; }
  const std::vector<Entry>& entries() const { return e_; }
  std::string report() const {
    double total = 0.0;
    for (auto const& x : e_) total += x.seconds;
    std::ostringstream ss;
    ss << std::fixed << std::setprecision(6);
    for (auto const& x : e_) {
      ss << x.name << ": " << x.seconds << "s calls=" << x.calls;
      if (total > 0.0) ss << " share=" << (100.0 * x.seconds / total) << "%";
      ss << '\n';
    }
    return ss.str();
  }

  class Scope {
    Profiler* p_;
    int id_;
    Clock::time_point begin_;
  public:
    Scope(Profiler& p, int id) : p_(&p), id_(id), begin_(Clock::now()) {}
    ~Scope() {
      p_->record(id_, std::chrono::duration<double>(Clock::now() - begin_).count());
    }
  };
};

} // namespace ahc

// ===== ahc/beam_select.hpp =====
#line 1 "ahc/beam_select.hpp"
#include <vector>
#include <unordered_set>
#include <unordered_map>
#include <algorithm>
#include <numeric>
#include <cstdint>
#include <cmath>

namespace ahc {

// Candidate-first Beam helper.
// Important pattern: generate only lightweight candidates, prune here, then materialize full states.
template<class Cand, class ScoreFn, class HashFn, class BucketFn>
std::vector<int> select_beam_candidates(
    const std::vector<Cand>& cands,
    int width,
    int per_bucket_cap,
    ScoreFn score_of,
    HashFn hash_of,
    BucketFn bucket_of,
    bool dedup_hash = true) {

  std::vector<int> order(cands.size());
  std::iota(order.begin(), order.end(), 0);
  auto cmp = [&](int a, int b) {
    const auto sa = score_of(cands[a]);
    const auto sb = score_of(cands[b]);
    if (sa != sb) return sa < sb;
    return a < b;
  };
  if (static_cast<int>(order.size()) > width * 4) {
    const int keep = std::min<int>(order.size(), width * 4);
    std::nth_element(order.begin(), order.begin() + keep, order.end(), cmp);
    order.resize(keep);
  }
  std::sort(order.begin(), order.end(), cmp);

  std::unordered_set<uint64_t> seen;
  if (dedup_hash) seen.reserve(width * 2 + 1);
  std::unordered_map<long long,int> bucket_count;
  if (per_bucket_cap > 0) bucket_count.reserve(width * 2 + 1);

  std::vector<int> selected;
  selected.reserve(width);
  for (int idx : order) {
    if (dedup_hash && !seen.insert(static_cast<uint64_t>(hash_of(cands[idx]))).second) continue;
    if (per_bucket_cap > 0) {
      const long long b = static_cast<long long>(bucket_of(cands[idx]));
      int& cnt = bucket_count[b];
      if (cnt >= per_bucket_cap) continue;
      ++cnt;
    }
    selected.push_back(idx);
    if (static_cast<int>(selected.size()) == width) break;
  }
  return selected;
}


// Faster version for hot loops: reuses caller-owned fixed hash tables.
// seen_hash / bucket_count should be reserved to >= ~2*width before the search.
template<class Cand, class ScoreFn, class HashFn, class BucketFn>
std::vector<int> select_beam_candidates_fast(
    const std::vector<Cand>& cands,
    int width,
    int per_bucket_cap,
    ScoreFn score_of,
    HashFn hash_of,
    BucketFn bucket_of,
    FlatHashSet64& seen_hash,
    FlatHashCounter64& bucket_count,
    bool dedup_hash = true) {

  std::vector<int> order(cands.size());
  std::iota(order.begin(), order.end(), 0);
  auto cmp = [&](int a, int b) {
    const auto sa = score_of(cands[a]);
    const auto sb = score_of(cands[b]);
    if (sa != sb) return sa < sb;
    return a < b;
  };
  if (static_cast<int>(order.size()) > width * 4) {
    const int keep = std::min<int>(order.size(), width * 4);
    std::nth_element(order.begin(), order.begin() + keep, order.end(), cmp);
    order.resize(keep);
  }
  std::sort(order.begin(), order.end(), cmp);

  seen_hash.clear();
  bucket_count.clear();
  std::vector<int> selected;
  selected.reserve(width);
  for (int idx : order) {
    if (dedup_hash && !seen_hash.insert(static_cast<uint64_t>(hash_of(cands[idx])))) continue;
    if (per_bucket_cap > 0) {
      const uint64_t b = static_cast<uint64_t>(static_cast<long long>(bucket_of(cands[idx])));
      int& cnt = bucket_count.get(b);
      if (cnt >= per_bucket_cap) continue;
      ++cnt;
    }
    selected.push_back(idx);
    if (static_cast<int>(selected.size()) == width) break;
  }
  return selected;
}

// Adjust width from observed layer time and remaining budget.
inline int adaptive_beam_width(int current, double layer_sec, double remaining_sec,
                               int layers_left, int min_width, int max_width,
                               double response = 0.25) {
  if (layer_sec <= 1e-12 || layers_left <= 0) return current;
  const double target = remaining_sec / layer_sec * current / layers_left;
  const double ratio = std::max(0.1, target / std::max(1, current));
  int next = static_cast<int>(current * std::pow(ratio, response));
  return std::clamp(next, min_width, max_width);
}

} // namespace ahc

// ===== ahc/beam_workspace.hpp =====
#line 1 "ahc/beam_workspace.hpp"
#include <vector>
#include <algorithm>
#include <numeric>
#include <cstdint>

namespace ahc {

// Reusable Beam selection workspace: no per-layer order/selected allocations.
// If the first preselection is over-filtered by dedup/buckets, it automatically
// retries on the full candidate set instead of returning an unnecessarily thin beam.
class BeamWorkspace {
  std::vector<int> order_;
  std::vector<int> selected_;
  FlatHashSet64 seen_;
  FlatHashCounter64 buckets_;
public:
  explicit BeamWorkspace(size_t expected_width = 128)
      : seen_(std::max<size_t>(16, expected_width * 4)),
        buckets_(std::max<size_t>(16, expected_width * 4)) {
    order_.reserve(expected_width * 8);
    selected_.reserve(expected_width);
  }
  void reserve(size_t candidates, size_t width) {
    order_.reserve(candidates);
    selected_.reserve(width);
    if (seen_.capacity() < width * 4) seen_.reserve(width * 4);
    if (buckets_.capacity() < width * 4) buckets_.reserve(width * 4);
  }

  template<class Cand, class ScoreFn, class HashFn, class BucketFn>
  const std::vector<int>& select(const std::vector<Cand>& cands,
                                 int width,
                                 int per_bucket_cap,
                                 ScoreFn score_of,
                                 HashFn hash_of,
                                 BucketFn bucket_of,
                                 bool dedup_hash = true,
                                 int preselect_factor = 4) {
    order_.resize(cands.size());
    std::iota(order_.begin(), order_.end(), 0);
    selected_.clear();
    if (width <= 0 || cands.empty()) return selected_;

    auto cmp = [&](int a, int b) {
      const auto sa = score_of(cands[a]);
      const auto sb = score_of(cands[b]);
      if (sa != sb) return sa < sb;
      return a < b;
    };

    const int n = static_cast<int>(order_.size());
    const int pre = std::min(n, std::max(width, width * std::max(1, preselect_factor)));
    if (pre < n) {
      std::nth_element(order_.begin(), order_.begin() + pre, order_.end(), cmp);
      std::sort(order_.begin(), order_.begin() + pre, cmp);
    } else {
      std::sort(order_.begin(), order_.end(), cmp);
    }

    auto scan = [&](int limit) {
      seen_.clear();
      buckets_.clear();
      selected_.clear();
      for (int oi = 0; oi < limit; ++oi) {
        int idx = order_[oi];
        if (dedup_hash && !seen_.insert(static_cast<uint64_t>(hash_of(cands[idx])))) continue;
        if (per_bucket_cap > 0) {
          auto b = static_cast<uint64_t>(static_cast<long long>(bucket_of(cands[idx])));
          int& cnt = buckets_.get(b);
          if (cnt >= per_bucket_cap) continue;
          ++cnt;
        }
        selected_.push_back(idx);
        if (static_cast<int>(selected_.size()) == width) break;
      }
    };

    scan(pre);
    if (static_cast<int>(selected_.size()) < width && pre < n) {
      // Rare fallback. Re-sort all candidates so diversity filters can fill the width.
      std::sort(order_.begin(), order_.end(), cmp);
      scan(n);
    }
    return selected_;
  }
};

} // namespace ahc

// ===== ahc/stage.hpp =====
#line 1 "ahc/stage.hpp"
#include <vector>
#include <cstdint>
#include <cassert>
#include <algorithm>

namespace ahc {

// One-proposal transaction helper.
// Writes go to a staging value; commit() accepts, reset() rejects.
// This is intentionally simpler than a general rollback log.
template<class T>
class StageValue {
  T value_{};
  T staged_{};
  bool dirty_ = false;
public:
  StageValue() = default;
  explicit StageValue(const T& v) : value_(v), staged_(v) {}

  [[nodiscard]] const T& get() const { return dirty_ ? staged_ : value_; }
  operator const T&() const { return get(); }

  void set(const T& v) { staged_ = v; dirty_ = true; }
  template<class U> void add(const U& x) { set(get() + x); }
  [[nodiscard]] bool dirty() const { return dirty_; }
  void commit() { if (dirty_) value_ = staged_; dirty_ = false; }
  void reset() { dirty_ = false; }
};

// One-level transactional array. Only touched indices are processed on commit;
// reject/reset is O(number of touched indices) only to invalidate bookkeeping,
// while values themselves never need to be restored.
template<class T>
class StageArray {
  std::vector<T> value_, staged_;
  std::vector<uint32_t> mark_;
  std::vector<int> touched_;
  uint32_t epoch_ = 1;

  void next_epoch() {
    ++epoch_;
    if (epoch_ == 0) {
      std::fill(mark_.begin(), mark_.end(), 0);
      epoch_ = 1;
    }
    touched_.clear();
  }
public:
  StageArray() = default;
  explicit StageArray(int n, const T& init = T()) { assign(n, init); }

  void assign(int n, const T& init = T()) {
    value_.assign(n, init);
    staged_.resize(n);
    mark_.assign(n, 0);
    touched_.clear();
    epoch_ = 1;
  }
  [[nodiscard]] int size() const { return static_cast<int>(value_.size()); }
  [[nodiscard]] bool dirty(int i) const { return mark_[i] == epoch_; }
  [[nodiscard]] const T& get(int i) const {
    assert(0 <= i && i < size());
    return dirty(i) ? staged_[i] : value_[i];
  }
  void set(int i, const T& v) {
    assert(0 <= i && i < size());
    if (!dirty(i)) {
      mark_[i] = epoch_;
      touched_.push_back(i);
    }
    staged_[i] = v;
  }
  template<class U> void add(int i, const U& x) { set(i, get(i) + x); }

  void commit() {
    for (int i : touched_) value_[i] = staged_[i];
    next_epoch();
  }
  void reset() { next_epoch(); }
  [[nodiscard]] const std::vector<int>& touched() const { return touched_; }

  struct Proxy {
    StageArray* self;
    int i;
    operator T() const { return self->get(i); }
    Proxy& operator=(const T& v) { self->set(i, v); return *this; }
  };
  Proxy operator[](int i) { return Proxy{this, i}; }
  const T& operator[](int i) const { return get(i); }
};

} // namespace ahc

// ===== ahc/index_set.hpp =====
#line 1 "ahc/index_set.hpp"
#include <vector>
#include <cstdint>
#include <cassert>
#include <algorithm>

namespace ahc {

// Dense/sparse integer set for [0,n). insert/erase/contains/random-index are O(1).
// clear() is O(1) via epoch stamping.
class IndexSet {
  std::vector<int> dense_;
  std::vector<int> pos_;
  std::vector<uint32_t> stamp_;
  uint32_t epoch_ = 1;
public:
  IndexSet() = default;
  explicit IndexSet(int n) { reset_universe(n); }

  void reset_universe(int n) {
    dense_.clear();
    dense_.reserve(n);
    pos_.assign(n, -1);
    stamp_.assign(n, 0);
    epoch_ = 1;
  }
  [[nodiscard]] int universe_size() const { return static_cast<int>(pos_.size()); }
  [[nodiscard]] int size() const { return static_cast<int>(dense_.size()); }
  [[nodiscard]] bool empty() const { return dense_.empty(); }
  [[nodiscard]] bool contains(int x) const {
    assert(0 <= x && x < universe_size());
    return stamp_[x] == epoch_;
  }
  bool insert(int x) {
    assert(0 <= x && x < universe_size());
    if (contains(x)) return false;
    stamp_[x] = epoch_;
    pos_[x] = size();
    dense_.push_back(x);
    return true;
  }
  bool erase(int x) {
    assert(0 <= x && x < universe_size());
    if (!contains(x)) return false;
    int p = pos_[x];
    int y = dense_.back();
    dense_[p] = y;
    pos_[y] = p;
    dense_.pop_back();
    stamp_[x] = 0;
    pos_[x] = -1;
    return true;
  }
  void clear() {
    dense_.clear();
    ++epoch_;
    if (epoch_ == 0) {
      std::fill(stamp_.begin(), stamp_.end(), 0);
      epoch_ = 1;
    }
  }
  [[nodiscard]] const std::vector<int>& items() const { return dense_; }
  [[nodiscard]] int operator[](int i) const { return dense_[i]; }

  template<class Rng>
  int random(Rng& rng) const {
    assert(!dense_.empty());
    return dense_[rng.uniform_int(0, size() - 1)];
  }
};

} // namespace ahc

// ===== ahc/cutoff_topk.hpp =====
#line 1 "ahc/cutoff_topk.hpp"
#include <vector>
#include <algorithm>
#include <cassert>
#include <functional>

namespace ahc {

// Keep roughly the best K values while accepting a large stream of candidates.
// Rather than maintaining a heap for every push, compact only when the buffer
// grows to cutoff_factor*K. Good for Beam candidate collection.
template<class T, class Better = std::less<T>>
class CutoffTopK {
  int k_ = 0;
  int cutoff_factor_ = 2;
  Better better_{};
  std::vector<T> v_;

  void compact() {
    if (k_ <= 0) { v_.clear(); return; }
    if (static_cast<int>(v_.size()) <= k_) return;
    auto mid = v_.begin() + k_;
    std::nth_element(v_.begin(), mid, v_.end(), better_);
    v_.resize(k_);
  }
public:
  CutoffTopK() = default;
  explicit CutoffTopK(int k, Better better = Better(), int cutoff_factor = 2)
      : k_(k), cutoff_factor_(std::max(2, cutoff_factor)), better_(std::move(better)) {
    v_.reserve(std::max(0, k_) * cutoff_factor_);
  }
  void reset(int k) {
    k_ = k;
    v_.clear();
    if (static_cast<int>(v_.capacity()) < k_ * cutoff_factor_) v_.reserve(k_ * cutoff_factor_);
  }
  void clear() { v_.clear(); }
  [[nodiscard]] int limit() const { return k_; }
  [[nodiscard]] int size() const { return static_cast<int>(v_.size()); }

  void push(const T& x) {
    if (k_ <= 0) return;
    v_.push_back(x);
    if (static_cast<int>(v_.size()) >= k_ * cutoff_factor_) compact();
  }
  void push(T&& x) {
    if (k_ <= 0) return;
    v_.push_back(std::move(x));
    if (static_cast<int>(v_.size()) >= k_ * cutoff_factor_) compact();
  }
  const std::vector<T>& finish(bool sort_result = true) {
    compact();
    if (sort_result) std::sort(v_.begin(), v_.end(), better_);
    return v_;
  }
  std::vector<T>& raw() { return v_; }
};

} // namespace ahc

// ===== ahc/anytime.hpp =====
#line 1 "ahc/anytime.hpp"
#include <vector>
#include <utility>
#include <ostream>
#include <iomanip>
#include <type_traits>
#include <functional>

namespace ahc {

// Records how solution quality improves over wall-clock time.
// Improvement events are always stored; sample() adds fixed-interval points.
template<class Score>
class AnytimeTrace {
public:
  struct Point { double sec; Score score; };
private:
  const Timer* timer_ = nullptr;
  double interval_ = 0.01;
  double next_sample_ = 0.0;
  bool has_best_ = false;
  Score best_{};
  std::vector<Point> points_;
public:
  AnytimeTrace() = default;
  AnytimeTrace(const Timer& timer, double interval_sec = 0.01)
      : timer_(&timer), interval_(interval_sec), next_sample_(0.0) {}

  template<class Better>
  bool update(const Score& score, Better better) {
    if (!timer_) return false;
    if (!has_best_ || better(score, best_)) {
      best_ = score;
      has_best_ = true;
      points_.push_back({timer_->elapsed(), score});
      return true;
    }
    return false;
  }
  template<class Better = std::less<Score>>
  bool update(const Score& score) { return update(score, Better{}); }

  void sample(const Score& score) {
    if (!timer_) return;
    double now = timer_->elapsed();
    if (now + 1e-15 >= next_sample_) {
      points_.push_back({now, score});
      do next_sample_ += interval_; while (next_sample_ <= now);
    }
  }
  [[nodiscard]] const std::vector<Point>& points() const { return points_; }
  void clear() { points_.clear(); has_best_ = false; next_sample_ = 0.0; }

  void write_csv(std::ostream& os) const {
    os << "sec,score\n";
    os << std::setprecision(17);
    for (const auto& p : points_) os << p.sec << ',' << p.score << '\n';
  }
};

} // namespace ahc

// ===== ahc/csr.hpp =====
#line 1 "ahc/csr.hpp"
#include <vector>
#include <cassert>
#include <utility>

namespace ahc {

// Minimal CSR for static adjacency. Build once, iterate cache-friendly.
template<class Edge = int>
class CSR {
  std::vector<int> start_;
  std::vector<Edge> elist_;
public:
  CSR() = default;

  template<class PairRange>
  CSR(int n, const PairRange& edges) { build(n, edges); }

  template<class PairRange>
  void build(int n, const PairRange& edges) {
    start_.assign(n + 1, 0);
    for (const auto& e : edges) {
      int from = e.first;
      assert(0 <= from && from < n);
      ++start_[from + 1];
    }
    for (int i = 1; i <= n; ++i) start_[i] += start_[i - 1];
    auto counter = start_;
    elist_.resize(edges.size());
    for (const auto& e : edges) elist_[counter[e.first]++] = e.second;
  }

  struct Range {
    const Edge* b;
    const Edge* e;
    const Edge* begin() const { return b; }
    const Edge* end() const { return e; }
    int size() const { return static_cast<int>(e - b); }
    const Edge& operator[](int i) const { return b[i]; }
  };

  [[nodiscard]] int n() const { return start_.empty() ? 0 : static_cast<int>(start_.size()) - 1; }
  [[nodiscard]] int m() const { return static_cast<int>(elist_.size()); }
  [[nodiscard]] Range operator[](int v) const {
    assert(0 <= v && v < n());
    return Range{elist_.data() + start_[v], elist_.data() + start_[v + 1]};
  }
};

} // namespace ahc

// ===== ahc/ring_queue.hpp =====
#line 1 "ahc/ring_queue.hpp"
#include <vector>
#include <cassert>
#include <algorithm>

namespace ahc {

// Reusable queue for BFS; avoids repeated deque allocations.
template<class T>
class RingQueue {
  std::vector<T> buf_;
  size_t head_ = 0, tail_ = 0, size_ = 0;
  void grow() {
    size_t oldcap = buf_.size();
    size_t newcap = std::max<size_t>(8, oldcap * 2);
    std::vector<T> nb(newcap);
    for (size_t i = 0; i < size_; ++i) nb[i] = std::move(buf_[(head_ + i) % oldcap]);
    buf_.swap(nb);
    head_ = 0; tail_ = size_;
  }
public:
  RingQueue() : buf_(8) {}
  explicit RingQueue(size_t cap) : buf_(std::max<size_t>(1, cap)) {}
  void clear() { head_ = tail_ = size_ = 0; }
  [[nodiscard]] bool empty() const { return size_ == 0; }
  [[nodiscard]] size_t size() const { return size_; }
  T& front() { assert(size_); return buf_[head_]; }
  const T& front() const { assert(size_); return buf_[head_]; }
  void push(const T& x) { if (size_ == buf_.size()) grow(); buf_[tail_] = x; tail_ = (tail_ + 1) % buf_.size(); ++size_; }
  void push(T&& x) { if (size_ == buf_.size()) grow(); buf_[tail_] = std::move(x); tail_ = (tail_ + 1) % buf_.size(); ++size_; }
  T pop() { assert(size_); T x = std::move(buf_[head_]); head_ = (head_ + 1) % buf_.size(); --size_; return x; }
};

} // namespace ahc

// ===== ahc/reversible_tree.hpp =====
#line 1 "ahc/reversible_tree.hpp"
#include <vector>
#include <utility>
#include <cassert>

namespace ahc {

// Small building block for copy-free tree/Beam traversal.
// Store moves as edges, then DFS with apply(move)/undo(move). Only one mutable
// problem State is needed while visiting the whole selected tree.
template<class Move>
class ReversibleMoveTree {
public:
  struct Node {
    int parent = -1;
    Move move{};
    int first_child = -1;
    int next_sibling = -1;
  };
private:
  std::vector<Node> nodes_;
public:
  ReversibleMoveTree() { nodes_.push_back(Node{}); }
  void clear() { nodes_.clear(); nodes_.push_back(Node{}); }
  [[nodiscard]] int root() const { return 0; }
  [[nodiscard]] int size() const { return static_cast<int>(nodes_.size()); }
  [[nodiscard]] const Node& operator[](int i) const { return nodes_[i]; }

  int add_child(int parent, const Move& move) {
    assert(0 <= parent && parent < size());
    int id = size();
    Node n; n.parent = parent; n.move = move; n.next_sibling = nodes_[parent].first_child;
    nodes_.push_back(std::move(n));
    nodes_[parent].first_child = id;
    return id;
  }

  template<class Apply, class Undo, class Visit>
  void dfs(Apply&& apply, Undo&& undo, Visit&& visit, int root_id = 0) const {
    assert(0 <= root_id && root_id < size());
    struct Frame { int node; int child; bool entered; };
    std::vector<Frame> st;
    st.reserve(nodes_.size());
    st.push_back({root_id, nodes_[root_id].first_child, false});
    visit(root_id);
    while (!st.empty()) {
      auto& f = st.back();
      if (f.child == -1) {
        int u = f.node;
        st.pop_back();
        if (!st.empty()) undo(nodes_[u].move);
        continue;
      }
      int v = f.child;
      f.child = nodes_[v].next_sibling;
      apply(nodes_[v].move);
      visit(v);
      st.push_back({v, nodes_[v].first_child, false});
    }
  }
};

} // namespace ahc

// ===== ahc/static_bitset.hpp =====
#line 1 "ahc/static_bitset.hpp"
#include <array>
#include <cstdint>
#include <cassert>
#include <bit>
#include <algorithm>

namespace ahc {

template<int N>
class StaticBitset {
  static_assert(N > 0);
  static constexpr int B = (N + 63) / 64;
  std::array<uint64_t, B> w_{};
  static constexpr uint64_t last_mask() {
    if constexpr (N % 64 == 0) return ~uint64_t(0);
    else return (uint64_t(1) << (N % 64)) - 1;
  }
public:
  void clear() { w_.fill(0); }
  void fill() { w_.fill(~uint64_t(0)); w_[B-1] &= last_mask(); }
  void set(int i) { assert(0 <= i && i < N); w_[i>>6] |= uint64_t(1) << (i&63); }
  void reset(int i) { assert(0 <= i && i < N); w_[i>>6] &= ~(uint64_t(1) << (i&63)); }
  void flip(int i) { assert(0 <= i && i < N); w_[i>>6] ^= uint64_t(1) << (i&63); }
  [[nodiscard]] bool test(int i) const { assert(0 <= i && i < N); return (w_[i>>6] >> (i&63)) & 1ULL; }
  [[nodiscard]] int count() const { int s=0; for (uint64_t x:w_) s += std::popcount(x); return s; }
  [[nodiscard]] bool any() const { for (uint64_t x:w_) if (x) return true; return false; }
  [[nodiscard]] bool none() const { return !any(); }
  [[nodiscard]] bool intersects(const StaticBitset& o) const { for(int i=0;i<B;++i) if(w_[i]&o.w_[i]) return true; return false; }
  StaticBitset& operator|=(const StaticBitset& o){ for(int i=0;i<B;++i) w_[i]|=o.w_[i]; return *this; }
  StaticBitset& operator&=(const StaticBitset& o){ for(int i=0;i<B;++i) w_[i]&=o.w_[i]; return *this; }
  StaticBitset& operator^=(const StaticBitset& o){ for(int i=0;i<B;++i) w_[i]^=o.w_[i]; w_[B-1]&=last_mask(); return *this; }
  friend StaticBitset operator|(StaticBitset a,const StaticBitset& b){ return a|=b; }
  friend StaticBitset operator&(StaticBitset a,const StaticBitset& b){ return a&=b; }
  friend StaticBitset operator^(StaticBitset a,const StaticBitset& b){ return a^=b; }
  [[nodiscard]] const auto& words() const { return w_; }
};

} // namespace ahc

// ===== ahc/search_loop.hpp =====
#line 1 "ahc/search_loop.hpp"

namespace ahc {

// Tiny helper for budgeted local-search loops. Body returns false to stop early.
template<class F>
unsigned long long run_until(Timer& timer, F&& body, unsigned check_mask = 255) {
  unsigned long long it = 0;
  for (;; ++it) {
    if ((it & check_mask) == 0 && timer.expired()) break;
    if (!body(it, timer.progress())) break;
  }
  return it;
}

} // namespace ahc

// ===== ahc/transposition_table.hpp =====
#line 1 "ahc/transposition_table.hpp"
#include <vector>
#include <cstdint>
#include <limits>
#include <algorithm>
#include <cassert>
#include <tuple>

namespace ahc {

// Fast approximate 2-way set-associative transposition table.
// Designed for Beam/DFS where an occasional collision-based replacement is acceptable.
// Lower score is assumed better by default.
template<class Score = long long>
class TranspositionTable2Way {
public:
  struct Entry {
    uint64_t key = 0;
    Score score{};
    int depth = -1;
    uint32_t age = 0;
    bool used = false;
  };

private:
  std::vector<Entry> tab_;
  size_t sets_ = 0;
  uint32_t age_ = 1;

  static size_t ceil_pow2(size_t x) {
    size_t p = 1;
    while (p < x) p <<= 1;
    return p;
  }

  size_t base(uint64_t key) const {
    return (static_cast<size_t>(mix64(key)) & (sets_ - 1)) * 2;
  }

public:
  TranspositionTable2Way() { reserve(1024); }
  explicit TranspositionTable2Way(size_t expected_entries) { reserve(expected_entries); }

  void reserve(size_t expected_entries) {
    sets_ = ceil_pow2(std::max<size_t>(8, (expected_entries + 1) / 2));
    tab_.assign(sets_ * 2, Entry{});
    age_ = 1;
  }

  void next_age() {
    ++age_;
    if (age_ == 0) {
      for (auto& e : tab_) e.age = 0;
      age_ = 1;
    }
  }

  void clear() {
    for (auto& e : tab_) e.used = false;
    age_ = 1;
  }

  [[nodiscard]] size_t capacity() const { return tab_.size(); }

  const Entry* find(uint64_t key) const {
    size_t b = base(key);
    for (int i = 0; i < 2; ++i) {
      const Entry& e = tab_[b + i];
      if (e.used && e.key == key) return &e;
    }
    return nullptr;
  }

  // Returns true when the caller should keep/expand this state.
  // A duplicate key is kept only when it improves score, or reaches the same score deeper
  // if prefer_deeper is enabled. New entries use a simple replacement priority:
  // stale first, then shallower, then worse score.
  bool accept(uint64_t key, Score score, int depth, bool prefer_deeper = true) {
    size_t b = base(key);
    for (int i = 0; i < 2; ++i) {
      Entry& e = tab_[b + i];
      if (e.used && e.key == key) {
        bool better = score < e.score;
        bool same_but_deeper = prefer_deeper && score == e.score && depth > e.depth;
        if (!better && !same_but_deeper) return false;
        e.score = score;
        e.depth = depth;
        e.age = age_;
        return true;
      }
    }

    for (int i = 0; i < 2; ++i) {
      Entry& e = tab_[b + i];
      if (!e.used) {
        e = Entry{key, score, depth, age_, true};
        return true;
      }
    }

    int victim = 0;
    // Prefer replacing stale; otherwise shallower/worse entries.
    const Entry& e0 = tab_[b];
    const Entry& e1 = tab_[b + 1];
    if ((e1.age != age_) != (e0.age != age_)) victim = (e1.age != age_) ? 1 : 0;
    else if (e1.depth < e0.depth) victim = 1;
    else if (e1.depth == e0.depth && e1.score > e0.score) victim = 1;

    tab_[b + victim] = Entry{key, score, depth, age_, true};
    return true;
  }
};

} // namespace ahc

// ===== ahc/axis_beam.hpp =====
#line 1 "ahc/axis_beam.hpp"
#include <vector>
#include <algorithm>
#include <cassert>

namespace ahc {

// Frontier storage for chokudai / skip-beam style searches.
// Each integer axis owns an independent bounded candidate set.
// Transitions may push directly to axis+1, axis+L, remaining-task count, time, etc.
template<class T, class Better = std::less<T>>
class AxisBeam {
  std::vector<CutoffTopK<T, Better>> beams_;
  int width_ = 0;
  Better better_{};
public:
  AxisBeam() = default;
  AxisBeam(int axes, int width, Better better = Better()) { reset(axes, width, std::move(better)); }

  void reset(int axes, int width, Better better = Better()) {
    assert(axes >= 0 && width >= 0);
    width_ = width;
    better_ = std::move(better);
    beams_.clear();
    beams_.reserve(axes);
    for (int i = 0; i < axes; ++i) beams_.emplace_back(width_, better_);
  }

  [[nodiscard]] int axes() const { return static_cast<int>(beams_.size()); }
  [[nodiscard]] int width() const { return width_; }
  void clear_axis(int axis) { beams_[axis].clear(); }
  void clear_all() { for (auto& b : beams_) b.clear(); }

  void push(int axis, const T& x) {
    assert(0 <= axis && axis < axes());
    beams_[axis].push(x);
  }
  void push(int axis, T&& x) {
    assert(0 <= axis && axis < axes());
    beams_[axis].push(std::move(x));
  }

  const std::vector<T>& finish(int axis, bool sort_result = true) {
    assert(0 <= axis && axis < axes());
    return beams_[axis].finish(sort_result);
  }

  std::vector<T>& raw(int axis) {
    assert(0 <= axis && axis < axes());
    return beams_[axis].raw();
  }
};

} // namespace ahc

// ===== ahc/work_budget.hpp =====
#line 1 "ahc/work_budget.hpp"
#include <algorithm>
#include <cmath>

namespace ahc {

// Exponential moving estimate of work throughput.
// Useful for adapting Beam width, number of repair trials, rollouts, etc.
class WorkBudgetController {
  double sec_per_unit_ = 0.0;
  double alpha_ = 0.25;
  bool ready_ = false;
public:
  explicit WorkBudgetController(double alpha = 0.25) : alpha_(std::clamp(alpha, 0.001, 1.0)) {}

  void observe(long long units, double seconds) {
    if (units <= 0 || seconds <= 0.0) return;
    double x = seconds / static_cast<double>(units);
    if (!ready_) {
      sec_per_unit_ = x;
      ready_ = true;
    } else {
      sec_per_unit_ = (1.0 - alpha_) * sec_per_unit_ + alpha_ * x;
    }
  }

  [[nodiscard]] bool ready() const { return ready_; }
  [[nodiscard]] double seconds_per_unit() const { return sec_per_unit_; }
  [[nodiscard]] double units_per_second() const { return ready_ && sec_per_unit_ > 0 ? 1.0 / sec_per_unit_ : 0.0; }

  // Choose work for one remaining stage assuming equal time share among remaining stages.
  [[nodiscard]] int recommend(double remaining_seconds,
                              int remaining_stages,
                              int min_units,
                              int max_units,
                              double utilization = 0.85) const {
    if (!ready_ || sec_per_unit_ <= 0.0) return std::clamp(min_units, min_units, max_units);
    remaining_stages = std::max(1, remaining_stages);
    utilization = std::clamp(utilization, 0.05, 1.0);
    double budget = std::max(0.0, remaining_seconds) * utilization / remaining_stages;
    long long x = static_cast<long long>(budget / sec_per_unit_);
    return static_cast<int>(std::clamp<long long>(x, min_units, max_units));
  }
};

} // namespace ahc

// ===== ahc/screening.hpp =====
#line 1 "ahc/screening.hpp"
#include <vector>
#include <algorithm>
#include <numeric>
#include <cassert>

namespace ahc {

// Two-stage candidate evaluation helper.
// First rank many candidates by a cheap score; only exact-evaluate the best M.
// This is intentionally minimal: exact evaluation remains problem-specific.
class ScreeningWorkspace {
  std::vector<int> order_;
public:
  void reserve(size_t n) { order_.reserve(n); }

  template<class Cand, class CheapScoreFn>
  const std::vector<int>& shortlist(const std::vector<Cand>& cands,
                                    int keep,
                                    CheapScoreFn cheap_score) {
    const int n = static_cast<int>(cands.size());
    keep = std::clamp(keep, 0, n);
    order_.resize(n);
    std::iota(order_.begin(), order_.end(), 0);
    auto cmp = [&](int a, int b) {
      auto sa = cheap_score(cands[a]);
      auto sb = cheap_score(cands[b]);
      if (sa != sb) return sa < sb;
      return a < b;
    };
    if (keep < n) {
      std::nth_element(order_.begin(), order_.begin() + keep, order_.end(), cmp);
      order_.resize(keep);
    }
    std::sort(order_.begin(), order_.end(), cmp);
    return order_;
  }
};

} // namespace ahc

// ===== ahc/pareto2d.hpp =====
#line 1 "ahc/pareto2d.hpp"
#include <vector>
#include <algorithm>
#include <numeric>

namespace ahc {

// Return indices of the non-dominated set for two minimization objectives.
// O(N log N). Useful as a diversity filter before scalar ranking.
template<class T, class F1, class F2>
std::vector<int> pareto_front_2d(const std::vector<T>& a, F1 f1, F2 f2) {
  std::vector<int> ord(a.size());
  std::iota(ord.begin(), ord.end(), 0);
  std::sort(ord.begin(), ord.end(), [&](int x, int y) {
    auto ax = f1(a[x]), ay = f1(a[y]);
    if (ax != ay) return ax < ay;
    auto bx = f2(a[x]), by = f2(a[y]);
    if (bx != by) return bx < by;
    return x < y;
  });

  std::vector<int> out;
  if (ord.empty()) return out;
  using V2 = decltype(f2(a[ord[0]]));
  bool have = false;
  V2 best2{};
  for (int id : ord) {
    V2 v = f2(a[id]);
    if (!have || v < best2) {
      out.push_back(id);
      best2 = v;
      have = true;
    }
  }
  return out;
}

} // namespace ahc

// ===== ahc/all.hpp =====
#line 1 "ahc/all.hpp"


// ===== ahc/object_pool.hpp =====
#line 1 "ahc/object_pool.hpp"
#include <vector>
#include <cstdint>
#include <cassert>
#include <utility>

namespace ahc {

// Stable integer handles with O(1) reuse. Useful for search nodes, temporary
// objects, and graph fragments where pointer stability is not required.
template<class T>
class ObjectPool {
  std::vector<T> data_;
  std::vector<int> free_;
  std::vector<uint8_t> alive_;
  int alive_count_ = 0;
public:
  ObjectPool() = default;
  explicit ObjectPool(size_t reserve_count) { reserve(reserve_count); }

  void reserve(size_t n) {
    data_.reserve(n);
    alive_.reserve(n);
    free_.reserve(n);
  }

  template<class... Args>
  int emplace(Args&&... args) {
    if (!free_.empty()) {
      int id = free_.back();
      free_.pop_back();
      data_[id] = T(std::forward<Args>(args)...);
      alive_[id] = 1;
      ++alive_count_;
      return id;
    }
    int id = static_cast<int>(data_.size());
    data_.emplace_back(std::forward<Args>(args)...);
    alive_.push_back(1);
    ++alive_count_;
    return id;
  }

  void erase(int id) {
    assert(0 <= id && id < static_cast<int>(data_.size()) && alive_[id]);
    alive_[id] = 0;
    free_.push_back(id);
    --alive_count_;
  }

  [[nodiscard]] bool alive(int id) const {
    return 0 <= id && id < static_cast<int>(alive_.size()) && alive_[id];
  }
  [[nodiscard]] int size() const { return alive_count_; }
  [[nodiscard]] int slots() const { return static_cast<int>(data_.size()); }
  [[nodiscard]] bool empty() const { return alive_count_ == 0; }

  T& operator[](int id) { assert(alive(id)); return data_[id]; }
  const T& operator[](int id) const { assert(alive(id)); return data_[id]; }

  void clear() {
    free_.clear();
    free_.reserve(data_.size());
    for (int i = static_cast<int>(data_.size()) - 1; i >= 0; --i) {
      if (alive_[i]) alive_[i] = 0;
      free_.push_back(i);
    }
    alive_count_ = 0;
  }
};

} // namespace ahc

// ===== ahc/reversible_path_arena.hpp =====
#line 1 "ahc/reversible_path_arena.hpp"
#include <vector>
#include <cassert>
#include <cstdint>
#include <algorithm>

namespace ahc {

// Persistent path arena plus an Euler-tour traversal of an arbitrary subset of
// leaves. The mutable problem state only needs apply(move)/undo(move).
// Every edge in the union of selected root-to-leaf paths is applied/undone at
// most once during traverse_subset().
template<class Move>
class ReversiblePathArena {
public:
  struct Node {
    int parent = -1;
    int depth = 0;
    Move move{};
  };

private:
  std::vector<Node> nodes_;
  std::vector<uint32_t> active_stamp_, target_stamp_;
  std::vector<int> first_child_, next_sibling_, active_nodes_;
  uint32_t stamp_ = 1;

  void ensure_workspace() {
    size_t n = nodes_.size();
    if (active_stamp_.size() < n) {
      active_stamp_.resize(n, 0);
      target_stamp_.resize(n, 0);
      first_child_.resize(n, -1);
      next_sibling_.resize(n, -1);
    }
  }

  void next_stamp() {
    ++stamp_;
    if (stamp_ == 0) {
      std::fill(active_stamp_.begin(), active_stamp_.end(), 0);
      std::fill(target_stamp_.begin(), target_stamp_.end(), 0);
      stamp_ = 1;
    }
  }

public:
  ReversiblePathArena() { nodes_.push_back(Node{}); }

  void clear() {
    nodes_.clear();
    nodes_.push_back(Node{});
    active_nodes_.clear();
  }
  void reserve(size_t n) {
    nodes_.reserve(n);
    active_nodes_.reserve(n);
  }

  [[nodiscard]] int root() const { return 0; }
  [[nodiscard]] int size() const { return static_cast<int>(nodes_.size()); }
  [[nodiscard]] const Node& operator[](int id) const { return nodes_[id]; }

  int add(int parent, const Move& move) {
    assert(0 <= parent && parent < size());
    int id = size();
    nodes_.push_back(Node{parent, nodes_[parent].depth + 1, move});
    return id;
  }
  int add(int parent, Move&& move) {
    assert(0 <= parent && parent < size());
    int id = size();
    nodes_.push_back(Node{parent, nodes_[parent].depth + 1, std::move(move)});
    return id;
  }

  std::vector<Move> restore(int id) const {
    assert(0 <= id && id < size());
    std::vector<Move> out;
    out.reserve(nodes_[id].depth);
    while (id != 0) {
      out.push_back(nodes_[id].move);
      id = nodes_[id].parent;
    }
    std::reverse(out.begin(), out.end());
    return out;
  }

  // Remove dead branches that do not lead to any current target. All targets
  // are remapped in-place, and every retained root-to-target move sequence is
  // preserved exactly. This is useful for long differential Beam runs where
  // old frontier branches would otherwise stay allocated forever.
  void compact_to_targets(std::vector<int>& targets) {
    if (targets.empty()) {
      clear();
      return;
    }
    ensure_workspace();
    next_stamp();
    active_stamp_[0] = stamp_;
    for (int t : targets) {
      assert(0 <= t && t < size());
      int u = t;
      while (u != 0 && active_stamp_[u] != stamp_) {
        active_stamp_[u] = stamp_;
        u = nodes_[u].parent;
      }
    }

    std::vector<int> remap(nodes_.size(), -1);
    std::vector<Node> kept;
    kept.reserve(nodes_.size());
    kept.push_back(Node{});
    remap[0] = 0;

    // Node ids are append-only, so every parent id is smaller than its child.
    for (int old_id = 1; old_id < size(); ++old_id) {
      if (active_stamp_[old_id] != stamp_) continue;
      int old_parent = nodes_[old_id].parent;
      assert(remap[old_parent] >= 0);
      int new_parent = remap[old_parent];
      remap[old_id] = static_cast<int>(kept.size());
      kept.push_back(Node{new_parent, kept[new_parent].depth + 1, nodes_[old_id].move});
    }
    for (int& t : targets) {
      assert(remap[t] >= 0);
      t = remap[t];
    }
    nodes_.swap(kept);

    // Workspace stamps index node ids, so reset after renumbering.
    active_stamp_.assign(nodes_.size(), 0);
    target_stamp_.assign(nodes_.size(), 0);
    first_child_.assign(nodes_.size(), -1);
    next_sibling_.assign(nodes_.size(), -1);
    active_nodes_.clear();
    stamp_ = 1;
  }

  template<class Apply, class Undo, class Visit>
  void traverse_subset(const std::vector<int>& targets,
                       Apply&& apply,
                       Undo&& undo,
                       Visit&& visit) {
    if (targets.empty()) return;
    ensure_workspace();
    next_stamp();
    active_nodes_.clear();

    // Mark the induced subtree (union of root-to-target paths).
    active_stamp_[0] = stamp_;
    active_nodes_.push_back(0);
    for (int t : targets) {
      assert(0 <= t && t < size());
      target_stamp_[t] = stamp_;
      int u = t;
      while (u != 0 && active_stamp_[u] != stamp_) {
        active_stamp_[u] = stamp_;
        active_nodes_.push_back(u);
        u = nodes_[u].parent;
      }
    }

    // Build temporary adjacency only for active nodes.
    for (int u : active_nodes_) first_child_[u] = -1;
    for (int u : active_nodes_) if (u != 0) {
      int p = nodes_[u].parent;
      next_sibling_[u] = first_child_[p];
      first_child_[p] = u;
    }

    struct Frame { int u; int child; };
    std::vector<Frame> st;
    st.reserve(active_nodes_.size());
    st.push_back({0, first_child_[0]});
    if (target_stamp_[0] == stamp_) visit(0);

    while (!st.empty()) {
      Frame& f = st.back();
      if (f.child == -1) {
        int u = f.u;
        st.pop_back();
        if (u != 0) undo(nodes_[u].move);
        continue;
      }
      int v = f.child;
      f.child = next_sibling_[v];
      apply(nodes_[v].move);
      if (target_stamp_[v] == stamp_) visit(v);
      st.push_back({v, first_child_[v]});
    }
  }
};

// Candidate-first differential Beam helper. The engine stores only persistent
// move paths; the user's mutable State is traversed with apply/undo. enumerate()
// is called while that State equals each frontier path and emits lightweight
// candidates. Selected candidates become the next frontier without State copies.
template<class Move, class Eval, class Better = std::less<Eval>>
class DifferentialBeamEngine {
public:
  struct Candidate {
    int parent_path = 0;
    Move move{};
    Eval eval{};
    uint64_t hash = 0;
    uint64_t bucket = 0;
  };

private:
  ReversiblePathArena<Move> paths_;
  std::vector<int> frontier_{0};
  std::vector<Candidate> candidates_;
  std::vector<int> next_frontier_;
  BeamWorkspace selector_;
  Better better_{};
  int steps_ = 0;
  int compact_interval_ = 0;
  size_t compact_node_limit_ = 0;

public:
  explicit DifferentialBeamEngine(size_t expected_width = 128, Better better = Better())
      : selector_(expected_width), better_(std::move(better)) {
    candidates_.reserve(expected_width * 16);
    next_frontier_.reserve(expected_width);
  }

  void clear() {
    paths_.clear();
    frontier_.assign(1, 0);
    candidates_.clear();
    next_frontier_.clear();
    steps_ = 0;
  }
  // 0 disables automatic compaction. A value such as 32 or 64 is usually a
  // good starting point when paths are long and Beam width is large.
  void set_compact_interval(int interval) { compact_interval_ = std::max(0, interval); }
  [[nodiscard]] int compact_interval() const { return compact_interval_; }
  // 0 disables the node-count trigger. If the live frontier itself needs more
  // nodes than this limit, compaction simply retains that live union.
  void set_compact_node_limit(size_t limit) { compact_node_limit_ = limit; }
  [[nodiscard]] size_t compact_node_limit() const { return compact_node_limit_; }
  void compact_now() { paths_.compact_to_targets(frontier_); }
  void reserve(size_t path_nodes, size_t candidates, size_t width) {
    paths_.reserve(path_nodes);
    candidates_.reserve(candidates);
    next_frontier_.reserve(width);
    selector_.reserve(candidates, width);
  }

  [[nodiscard]] const std::vector<int>& frontier() const { return frontier_; }
  [[nodiscard]] const ReversiblePathArena<Move>& paths() const { return paths_; }
  [[nodiscard]] ReversiblePathArena<Move>& paths() { return paths_; }

  template<class Apply, class Undo, class Enumerate>
  int step(int width,
           int per_bucket_cap,
           Apply&& apply,
           Undo&& undo,
           Enumerate&& enumerate,
           bool dedup_hash = true,
           int preselect_factor = 4) {
    candidates_.clear();

    paths_.traverse_subset(
      frontier_,
      std::forward<Apply>(apply),
      std::forward<Undo>(undo),
      [&](int path_id) {
        auto emit = [&](const Move& move, const Eval& eval, uint64_t hash, uint64_t bucket = 0) {
          candidates_.push_back(Candidate{path_id, move, eval, hash, bucket});
        };
        enumerate(path_id, emit);
      }
    );

    auto score_of = [](const Candidate& c) -> const Eval& { return c.eval; };
    // BeamWorkspace assumes smaller score. For custom Better, map by sorting ourselves.
    // std::less is the overwhelmingly common AHC case; custom Better uses a local order.
    std::vector<int> chosen;
    if constexpr (std::is_same_v<Better, std::less<Eval>>) {
      const auto& ids = selector_.select(
          candidates_, width, per_bucket_cap,
          score_of,
          [](const Candidate& c){ return c.hash; },
          [](const Candidate& c){ return c.bucket; },
          dedup_hash, preselect_factor);
      chosen.assign(ids.begin(), ids.end());
    } else {
      chosen.resize(candidates_.size());
      std::iota(chosen.begin(), chosen.end(), 0);
      std::sort(chosen.begin(), chosen.end(), [&](int a, int b) {
        if (better_(candidates_[a].eval, candidates_[b].eval)) return true;
        if (better_(candidates_[b].eval, candidates_[a].eval)) return false;
        return a < b;
      });
      FlatHashSet64 seen(std::max<size_t>(16, width * 4));
      FlatHashCounter64 buckets(std::max<size_t>(16, width * 4));
      std::vector<int> filtered;
      filtered.reserve(width);
      for (int id : chosen) {
        const auto& c = candidates_[id];
        if (dedup_hash && !seen.insert(c.hash)) continue;
        if (per_bucket_cap > 0) {
          int& cnt = buckets.get(c.bucket);
          if (cnt >= per_bucket_cap) continue;
          ++cnt;
        }
        filtered.push_back(id);
        if (static_cast<int>(filtered.size()) == width) break;
      }
      chosen.swap(filtered);
    }

    next_frontier_.clear();
    for (int id : chosen) {
      auto& c = candidates_[id];
      next_frontier_.push_back(paths_.add(c.parent_path, c.move));
    }
    frontier_.swap(next_frontier_);
    ++steps_;
    const bool by_interval = compact_interval_ > 0 && (steps_ % compact_interval_) == 0;
    const bool by_limit = compact_node_limit_ > 0 && static_cast<size_t>(paths_.size()) > compact_node_limit_;
    if (by_interval || by_limit) paths_.compact_to_targets(frontier_);
    return static_cast<int>(frontier_.size());
  }
};

} // namespace ahc

// ===== ahc/pareto_archive.hpp =====
#line 1 "ahc/pareto_archive.hpp"
#include <vector>
#include <array>
#include <algorithm>
#include <limits>
#include <cmath>
#include <cassert>
#include <cstdint>

namespace ahc {

// Small multi-objective elite archive (all objectives minimized).
// Dominated entries are removed. If capacity is exceeded, normalized NSGA-II
// style crowding distance removes the most redundant interior point.
template<class State, size_t D>
class ParetoEliteArchive {
public:
  using Objective = std::array<double, D>;
  struct Entry {
    State state;
    Objective obj{};
    uint64_t serial = 0;
  };

private:
  int capacity_ = 16;
  Objective eps_{};
  uint64_t serial_ = 0;
  std::vector<Entry> a_;

  bool dominates(const Objective& x, const Objective& y) const {
    bool strict = false;
    for (size_t i = 0; i < D; ++i) {
      if (x[i] > y[i] + eps_[i]) return false;
      if (x[i] < y[i] - eps_[i]) strict = true;
    }
    return strict;
  }

  bool equivalent(const Objective& x, const Objective& y) const {
    for (size_t i = 0; i < D; ++i)
      if (std::abs(x[i] - y[i]) > eps_[i]) return false;
    return true;
  }

  int most_crowded_index() const {
    const int n = static_cast<int>(a_.size());
    assert(n > 0);
    if (n <= 2) {
      int oldest = 0;
      for (int i = 1; i < n; ++i) if (a_[i].serial < a_[oldest].serial) oldest = i;
      return oldest;
    }
    std::vector<double> crowd(n, 0.0);
    std::vector<int> ord(n);
    std::iota(ord.begin(), ord.end(), 0);

    for (size_t d = 0; d < D; ++d) {
      std::sort(ord.begin(), ord.end(), [&](int x, int y) {
        if (a_[x].obj[d] != a_[y].obj[d]) return a_[x].obj[d] < a_[y].obj[d];
        return a_[x].serial < a_[y].serial;
      });
      double lo = a_[ord.front()].obj[d];
      double hi = a_[ord.back()].obj[d];
      crowd[ord.front()] = crowd[ord.back()] = std::numeric_limits<double>::infinity();
      double den = hi - lo;
      if (den <= 1e-15) continue;
      for (int k = 1; k + 1 < n; ++k) {
        int id = ord[k];
        crowd[id] += (a_[ord[k + 1]].obj[d] - a_[ord[k - 1]].obj[d]) / den;
      }
    }

    int victim = -1;
    for (int i = 0; i < n; ++i) {
      if (!std::isfinite(crowd[i])) continue;
      if (victim < 0 || crowd[i] < crowd[victim] - 1e-15 ||
          (std::abs(crowd[i] - crowd[victim]) <= 1e-15 && a_[i].serial < a_[victim].serial)) {
        victim = i;
      }
    }
    if (victim >= 0) return victim;
    // All points are extremes (possible for very small/degenerate archives).
    int oldest = 0;
    for (int i = 1; i < n; ++i) if (a_[i].serial < a_[oldest].serial) oldest = i;
    return oldest;
  }

public:
  explicit ParetoEliteArchive(int capacity = 16, Objective eps = {})
      : capacity_(std::max(1, capacity)), eps_(eps) {
    a_.reserve(capacity_ + 1);
  }

  [[nodiscard]] int size() const { return static_cast<int>(a_.size()); }
  [[nodiscard]] bool empty() const { return a_.empty(); }
  [[nodiscard]] const std::vector<Entry>& entries() const { return a_; }
  void clear() { a_.clear(); }

  bool insert(const State& state, const Objective& obj) {
    for (const auto& e : a_) {
      if (dominates(e.obj, obj) || equivalent(e.obj, obj)) return false;
    }

    a_.erase(std::remove_if(a_.begin(), a_.end(), [&](const Entry& e) {
      return dominates(obj, e.obj);
    }), a_.end());

    a_.push_back(Entry{state, obj, serial_++});
    if (static_cast<int>(a_.size()) > capacity_) {
      int victim = most_crowded_index();
      bool removed_new = (victim == static_cast<int>(a_.size()) - 1);
      a_.erase(a_.begin() + victim);
      return !removed_new;
    }
    return true;
  }

  const Entry& sample_uniform(Rng& rng) const {
    assert(!a_.empty());
    return a_[rng.int_range(0, size())];
  }

  // Choose the archive member minimizing a user-provided scalarization.
  template<class ScalarFn>
  const Entry& best_by(ScalarFn scalar) const {
    assert(!a_.empty());
    int b = 0;
    for (int i = 1; i < size(); ++i)
      if (scalar(a_[i].obj) < scalar(a_[b].obj)) b = i;
    return a_[b];
  }
};

} // namespace ahc

// ===== ahc/reheat_restart.hpp =====
#line 1 "ahc/reheat_restart.hpp"
#include <algorithm>

namespace ahc {

enum class SearchControlAction { none, reheat, restart };

// Window-based stagnation controller for SA/LNS. It does not own the solution;
// it only reports when to reheat or restart and exposes a temperature multiplier.
class ReheatRestartController {
public:
  struct Config {
    int window = 256;
    int stagnant_windows_for_reheat = 2;
    int reheats_before_restart = 2;
    double min_accept_rate = 0.02;
    double reheat_factor = 4.0;
    double max_temperature_scale = 64.0;
    double scale_decay_per_step = 0.999;
  };

private:
  Config cfg_{};
  int seen_ = 0, accepted_ = 0, improved_ = 0, best_ = 0;
  int stagnant_windows_ = 0, reheats_ = 0;
  double temp_scale_ = 1.0;

public:
  ReheatRestartController() : ReheatRestartController(Config{}) {}
  explicit ReheatRestartController(Config cfg) : cfg_(cfg) {
    cfg_.window = std::max(1, cfg_.window);
    cfg_.stagnant_windows_for_reheat = std::max(1, cfg_.stagnant_windows_for_reheat);
    cfg_.reheats_before_restart = std::max(0, cfg_.reheats_before_restart);
    cfg_.min_accept_rate = std::clamp(cfg_.min_accept_rate, 0.0, 1.0);
    cfg_.reheat_factor = std::max(1.0, cfg_.reheat_factor);
    cfg_.max_temperature_scale = std::max(1.0, cfg_.max_temperature_scale);
    cfg_.scale_decay_per_step = std::clamp(cfg_.scale_decay_per_step, 0.0, 1.0);
  }

  [[nodiscard]] double temperature_scale() const { return temp_scale_; }
  [[nodiscard]] double scale_temperature(double base) const { return base * temp_scale_; }
  [[nodiscard]] int reheat_count() const { return reheats_; }

  void reset_after_restart() {
    seen_ = accepted_ = improved_ = best_ = 0;
    stagnant_windows_ = reheats_ = 0;
    temp_scale_ = 1.0;
  }

  SearchControlAction observe(bool accepted, bool improved, bool new_best) {
    temp_scale_ = 1.0 + (temp_scale_ - 1.0) * cfg_.scale_decay_per_step;
    ++seen_;
    accepted_ += accepted;
    improved_ += improved;
    best_ += new_best;
    if (seen_ < cfg_.window) return SearchControlAction::none;

    double accept_rate = static_cast<double>(accepted_) / seen_;
    bool progress = best_ > 0 || improved_ > 0;
    if (progress) stagnant_windows_ = 0;
    else ++stagnant_windows_;

    seen_ = accepted_ = improved_ = best_ = 0;

    if (stagnant_windows_ < cfg_.stagnant_windows_for_reheat) return SearchControlAction::none;
    if (accept_rate > cfg_.min_accept_rate && stagnant_windows_ == cfg_.stagnant_windows_for_reheat)
      return SearchControlAction::none;

    stagnant_windows_ = 0;
    if (reheats_ < cfg_.reheats_before_restart) {
      ++reheats_;
      temp_scale_ = std::min(cfg_.max_temperature_scale,
                             std::max(temp_scale_, cfg_.reheat_factor));
      return SearchControlAction::reheat;
    }
    return SearchControlAction::restart;
  }
};

} // namespace ahc

// ===== ahc/bit_grid.hpp =====
#line 1 "ahc/bit_grid.hpp"
#include <array>
#include <cstdint>
#include <bit>
#include <cassert>
#include <algorithm>
#include <limits>

namespace ahc {

// Fixed-size bit-parallel grid. Index = r*W+c. Designed for small/medium AHC
// grids where obstacle propagation, reachability, neighborhood masks, or simple
// BFS layers dominate runtime.
template<int H, int W>
class BitGrid {
  static_assert(H > 0 && W > 0);
  static constexpr int N = H * W;
  static constexpr int B = (N + 63) / 64;
  std::array<uint64_t, B> w_{};

  static constexpr uint64_t last_mask() {
    if constexpr (N % 64 == 0) return ~uint64_t(0);
    else return (uint64_t(1) << (N % 64)) - 1;
  }

  BitGrid shift_linear(int d) const {
    BitGrid out;
    if (d == 0) return *this;
    if (d > 0) {
      int ws = d >> 6, bs = d & 63;
      for (int i = 0; i < B; ++i) if (w_[i]) {
        int j = i + ws;
        if (j < B) out.w_[j] |= w_[i] << bs;
        if (bs && j + 1 < B) out.w_[j + 1] |= w_[i] >> (64 - bs);
      }
    } else {
      d = -d;
      int ws = d >> 6, bs = d & 63;
      for (int i = 0; i < B; ++i) if (w_[i]) {
        int j = i - ws;
        if (j >= 0) out.w_[j] |= w_[i] >> bs;
        if (bs && j - 1 >= 0) out.w_[j - 1] |= w_[i] << (64 - bs);
      }
    }
    out.w_[B - 1] &= last_mask();
    return out;
  }

public:
  BitGrid() = default;

  void clear() { w_.fill(0); }
  void fill() { w_.fill(~uint64_t(0)); w_[B - 1] &= last_mask(); }
  void set(int r, int c) { assert(0 <= r && r < H && 0 <= c && c < W); set_index(r * W + c); }
  void reset(int r, int c) { assert(0 <= r && r < H && 0 <= c && c < W); reset_index(r * W + c); }
  [[nodiscard]] bool test(int r, int c) const { assert(0 <= r && r < H && 0 <= c && c < W); return test_index(r * W + c); }

  void set_index(int i) { assert(0 <= i && i < N); w_[i >> 6] |= uint64_t(1) << (i & 63); }
  void reset_index(int i) { assert(0 <= i && i < N); w_[i >> 6] &= ~(uint64_t(1) << (i & 63)); }
  [[nodiscard]] bool test_index(int i) const { assert(0 <= i && i < N); return (w_[i >> 6] >> (i & 63)) & 1ULL; }

  [[nodiscard]] int count() const { int s = 0; for (auto x : w_) s += std::popcount(x); return s; }
  [[nodiscard]] bool any() const { for (auto x : w_) if (x) return true; return false; }
  [[nodiscard]] bool none() const { return !any(); }

  [[nodiscard]] BitGrid north() const { return shift_linear(-W); }
  [[nodiscard]] BitGrid south() const { return shift_linear(+W); }
  [[nodiscard]] BitGrid east() const {
    BitGrid x = shift_linear(+1);
    for (int r = 0; r < H; ++r) x.reset_index(r * W); // remove row-wrap
    return x;
  }
  [[nodiscard]] BitGrid west() const {
    BitGrid x = shift_linear(-1);
    for (int r = 0; r < H; ++r) x.reset_index(r * W + (W - 1)); // remove row-wrap
    return x;
  }
  [[nodiscard]] BitGrid neighbors4() const { return north() | south() | east() | west(); }
  [[nodiscard]] BitGrid closed4() const { return *this | neighbors4(); }

  [[nodiscard]] bool intersects(const BitGrid& o) const {
    for (int i = 0; i < B; ++i) if (w_[i] & o.w_[i]) return true;
    return false;
  }

  BitGrid& operator|=(const BitGrid& o) { for (int i = 0; i < B; ++i) w_[i] |= o.w_[i]; return *this; }
  BitGrid& operator&=(const BitGrid& o) { for (int i = 0; i < B; ++i) w_[i] &= o.w_[i]; return *this; }
  BitGrid& operator^=(const BitGrid& o) { for (int i = 0; i < B; ++i) w_[i] ^= o.w_[i]; w_[B-1] &= last_mask(); return *this; }
  friend BitGrid operator|(BitGrid a, const BitGrid& b) { return a |= b; }
  friend BitGrid operator&(BitGrid a, const BitGrid& b) { return a &= b; }
  friend BitGrid operator^(BitGrid a, const BitGrid& b) { return a ^= b; }
  friend BitGrid operator~(BitGrid a) { for (auto& x : a.w_) x = ~x; a.w_[B-1] &= last_mask(); return a; }
  friend bool operator==(const BitGrid& a, const BitGrid& b) { return a.w_ == b.w_; }
  friend bool operator!=(const BitGrid& a, const BitGrid& b) { return !(a == b); }

  template<class F>
  void for_each(F&& f) const {
    for (int bi = 0; bi < B; ++bi) {
      uint64_t x = w_[bi];
      while (x) {
        int b = std::countr_zero(x);
        int id = bi * 64 + b;
        if (id < N) f(id / W, id % W);
        x &= x - 1;
      }
    }
  }

  [[nodiscard]] BitGrid flood_fill(const BitGrid& passable) const {
    BitGrid seen = *this & passable;
    for (;;) {
      BitGrid nxt = (seen | seen.neighbors4()) & passable;
      if (nxt == seen) return seen;
      seen = nxt;
    }
  }

  // Shortest 4-neighbor distance between any set bit in *this and target.
  // Returns -1 if unreachable.
  [[nodiscard]] int distance_to(const BitGrid& target, const BitGrid& passable,
                                int max_distance = N) const {
    BitGrid frontier = *this & passable;
    BitGrid seen = frontier;
    if (frontier.intersects(target)) return 0;
    for (int d = 1; d <= max_distance && frontier.any(); ++d) {
      frontier = frontier.neighbors4() & passable & ~seen;
      if (frontier.intersects(target)) return d;
      seen |= frontier;
    }
    return -1;
  }

  [[nodiscard]] const std::array<uint64_t, B>& words() const { return w_; }
};

} // namespace ahc

// ===== ahc/portfolio.hpp =====
#line 1 "ahc/portfolio.hpp"
#include <vector>
#include <cmath>
#include <algorithm>
#include <cassert>

namespace ahc {

// Time-aware multi-armed scheduler for heterogeneous search procedures such as
// SA / Beam / LNS / exact local search. The caller supplies a nonnegative reward
// (typically normalized score improvement) and the elapsed seconds of each run.
// Selection uses an EMA reward-rate plus a UCB-style exploration bonus.
class PortfolioScheduler {
public:
  struct Config {
    double ema_alpha = 0.25;
    double exploration = 0.35;
    double best_update_bonus = 0.0;
    double min_seconds = 1e-6;
  };
  struct ArmStats {
    int pulls = 0;
    int best_updates = 0;
    double seconds = 0.0;
    double total_reward = 0.0;
    double ema_reward_rate = 0.0;
  };

private:
  Config cfg_{};
  std::vector<ArmStats> arms_;
  int total_pulls_ = 0;
  double total_seconds_ = 0.0;

public:
  explicit PortfolioScheduler(int arms) : PortfolioScheduler(arms, Config{}) {}
  PortfolioScheduler(int arms, Config cfg) : cfg_(cfg), arms_(std::max(1, arms)) {
    cfg_.ema_alpha = std::clamp(cfg_.ema_alpha, 0.0, 1.0);
    cfg_.exploration = std::max(0.0, cfg_.exploration);
    cfg_.min_seconds = std::max(1e-12, cfg_.min_seconds);
  }

  [[nodiscard]] int size() const { return static_cast<int>(arms_.size()); }
  [[nodiscard]] int total_pulls() const { return total_pulls_; }
  [[nodiscard]] double total_seconds() const { return total_seconds_; }
  [[nodiscard]] const ArmStats& stats(int arm) const { assert(0 <= arm && arm < size()); return arms_[arm]; }

  int choose() const {
    for (int i = 0; i < size(); ++i) if (arms_[i].pulls == 0) return i;
    int best = 0;
    double best_score = -1e300;
    const double log_term = std::log(1.0 + std::max(1, total_pulls_));
    for (int i = 0; i < size(); ++i) {
      const auto& a = arms_[i];
      const double explore = cfg_.exploration * std::sqrt(log_term / std::max(1, a.pulls));
      const double score = a.ema_reward_rate + explore;
      if (score > best_score) {
        best_score = score;
        best = i;
      }
    }
    return best;
  }

  void observe(int arm, double seconds, double reward, bool new_global_best = false) {
    assert(0 <= arm && arm < size());
    auto& a = arms_[arm];
    seconds = std::max(cfg_.min_seconds, seconds);
    reward = std::max(0.0, reward) + (new_global_best ? cfg_.best_update_bonus : 0.0);
    const double rate = reward / seconds;
    if (a.pulls == 0) a.ema_reward_rate = rate;
    else a.ema_reward_rate = (1.0 - cfg_.ema_alpha) * a.ema_reward_rate + cfg_.ema_alpha * rate;
    ++a.pulls;
    ++total_pulls_;
    a.best_updates += new_global_best;
    a.seconds += seconds;
    a.total_reward += reward;
    total_seconds_ += seconds;
  }

  // A conservative slice suggestion. The chosen arm still owns its internal
  // stopping logic; this just prevents one arm from consuming the whole budget.
  [[nodiscard]] double suggested_slice(double remaining_seconds,
                                       double min_slice = 0.005,
                                       double max_slice = 0.20,
                                       double remaining_fraction = 0.08) const {
    if (remaining_seconds <= 0.0) return 0.0;
    double x = remaining_seconds * std::clamp(remaining_fraction, 0.0, 1.0);
    return std::min(remaining_seconds, std::clamp(x, min_slice, max_slice));
  }
};

} // namespace ahc

// ===== ahc/bulk_eval.hpp =====
#line 1 "ahc/bulk_eval.hpp"
#include <cstddef>
#include <cstdint>
#include <span>
#include <cmath>
#include <cassert>
#include <cstdlib>

namespace ahc {

// Small autovectorization-friendly kernels for hot evaluation loops. These have
// scalar semantics and require no ISA-specific compile flag; GCC/Clang can map
// them to SIMD under -O2/-O3 when profitable.
inline int64_t bulk_sum_i32(std::span<const int> a) {
  int64_t s = 0;
#if defined(__clang__)
#pragma clang loop vectorize(enable)
#elif defined(__GNUC__)
#pragma GCC ivdep
#endif
  for (size_t i = 0; i < a.size(); ++i) s += a[i];
  return s;
}

inline int64_t bulk_dot_i32(std::span<const int> a, std::span<const int> b) {
  assert(a.size() == b.size());
  int64_t s = 0;
#if defined(__clang__)
#pragma clang loop vectorize(enable)
#elif defined(__GNUC__)
#pragma GCC ivdep
#endif
  for (size_t i = 0; i < a.size(); ++i) s += int64_t(a[i]) * b[i];
  return s;
}

inline int64_t bulk_l1_i32(std::span<const int> a, std::span<const int> b) {
  assert(a.size() == b.size());
  int64_t s = 0;
#if defined(__clang__)
#pragma clang loop vectorize(enable)
#elif defined(__GNUC__)
#pragma GCC ivdep
#endif
  for (size_t i = 0; i < a.size(); ++i) s += std::llabs(int64_t(a[i]) - b[i]);
  return s;
}

inline float bulk_dot_f32(std::span<const float> a, std::span<const float> b) {
  assert(a.size() == b.size());
  float s = 0.0f;
#if defined(__clang__)
#pragma clang loop vectorize(enable)
#elif defined(__GNUC__)
#pragma GCC ivdep
#endif
  for (size_t i = 0; i < a.size(); ++i) s += a[i] * b[i];
  return s;
}

} // namespace ahc

// ===== ahc/svg.hpp =====
#line 1 "ahc/svg.hpp"
#include <ostream>
#include <string>
#include <string_view>
#include <vector>
#include <utility>
#include <sstream>
#include <iomanip>

namespace ahc {

// Tiny dependency-free SVG writer intended for LOCAL visual debugging.
class SvgWriter {
  std::ostream* os_ = nullptr;
  bool finished_ = false;

  static std::string esc(std::string_view s) {
    std::string out;
    out.reserve(s.size());
    for (char c : s) {
      if (c == '&') out += "&amp;";
      else if (c == '<') out += "&lt;";
      else if (c == '>') out += "&gt;";
      else if (c == '"') out += "&quot;";
      else out += c;
    }
    return out;
  }

public:
  SvgWriter(std::ostream& os, int width, int height, std::string_view background = "white")
      : os_(&os) {
    *os_ << "<svg xmlns=\"http://www.w3.org/2000/svg\" width=\"" << width
         << "\" height=\"" << height << "\" viewBox=\"0 0 " << width << ' ' << height << "\">\n";
    if (!background.empty()) rect(0, 0, width, height, background, "none", 0.0);
  }
  SvgWriter(const SvgWriter&) = delete;
  SvgWriter& operator=(const SvgWriter&) = delete;
  ~SvgWriter() { finish(); }

  void finish() {
    if (!finished_ && os_) {
      *os_ << "</svg>\n";
      finished_ = true;
    }
  }

  void rect(double x, double y, double w, double h,
            std::string_view fill = "none", std::string_view stroke = "black",
            double stroke_width = 1.0, double rx = 0.0) {
    *os_ << "<rect x=\"" << x << "\" y=\"" << y << "\" width=\"" << w
         << "\" height=\"" << h << "\" fill=\"" << fill << "\" stroke=\""
         << stroke << "\" stroke-width=\"" << stroke_width << "\"";
    if (rx > 0) *os_ << " rx=\"" << rx << "\"";
    *os_ << "/>\n";
  }

  void circle(double cx, double cy, double r,
              std::string_view fill = "none", std::string_view stroke = "black",
              double stroke_width = 1.0) {
    *os_ << "<circle cx=\"" << cx << "\" cy=\"" << cy << "\" r=\"" << r
         << "\" fill=\"" << fill << "\" stroke=\"" << stroke
         << "\" stroke-width=\"" << stroke_width << "\"/>\n";
  }

  void line(double x1, double y1, double x2, double y2,
            std::string_view stroke = "black", double stroke_width = 1.0,
            std::string_view extra = "") {
    *os_ << "<line x1=\"" << x1 << "\" y1=\"" << y1 << "\" x2=\"" << x2
         << "\" y2=\"" << y2 << "\" stroke=\"" << stroke
         << "\" stroke-width=\"" << stroke_width << "\"";
    if (!extra.empty()) *os_ << ' ' << extra;
    *os_ << "/>\n";
  }

  void text(double x, double y, std::string_view value,
            double font_size = 12.0, std::string_view fill = "black",
            std::string_view anchor = "middle") {
    *os_ << "<text x=\"" << x << "\" y=\"" << y << "\" font-size=\""
         << font_size << "\" fill=\"" << fill << "\" text-anchor=\"" << anchor
         << "\" dominant-baseline=\"middle\">" << esc(value) << "</text>\n";
  }

  template<class Range>
  void polyline(const Range& pts, std::string_view stroke = "black",
                double stroke_width = 1.0, std::string_view fill = "none") {
    *os_ << "<polyline points=\"";
    bool first = true;
    for (auto&& p : pts) {
      if (!first) *os_ << ' ';
      first = false;
      *os_ << p.first << ',' << p.second;
    }
    *os_ << "\" fill=\"" << fill << "\" stroke=\"" << stroke
         << "\" stroke-width=\"" << stroke_width << "\"/>\n";
  }

  void begin_group(std::string_view attrs = "") {
    *os_ << "<g";
    if (!attrs.empty()) *os_ << ' ' << attrs;
    *os_ << ">\n";
  }
  void end_group() { *os_ << "</g>\n"; }
};

} // namespace ahc

// ===== ahc/params.hpp =====
#line 1 "ahc/params.hpp"
#include <string>
#include <string_view>
#include <vector>
#include <functional>
#include <sstream>
#include <fstream>
#include <ostream>
#include <type_traits>
#include <algorithm>
#include <cctype>

namespace ahc {

// Lightweight parameter registry for LOCAL sweeps. Bind existing variables,
// then override them from --key=value / --key value or a key=value file.
class ParamRegistry {
  struct Entry {
    std::string name;
    std::function<bool(std::string_view)> set;
    std::function<std::string()> get;
  };
  std::vector<Entry> e_;

  static std::string trim(std::string s) {
    auto ws=[](unsigned char c){return std::isspace(c);};
    while(!s.empty() && ws(s.front())) s.erase(s.begin());
    while(!s.empty() && ws(s.back())) s.pop_back();
    return s;
  }

  template<class T>
  static bool parse(std::string_view sv, T& out) {
    if constexpr (std::is_same_v<T,std::string>) {
      out=std::string(sv); return true;
    } else if constexpr (std::is_same_v<T,bool>) {
      std::string s(sv);
      std::transform(s.begin(),s.end(),s.begin(),[](unsigned char c){return char(std::tolower(c));});
      if(s=="1"||s=="true"||s=="yes"||s=="on"){out=true;return true;}
      if(s=="0"||s=="false"||s=="no"||s=="off"){out=false;return true;}
      return false;
    } else {
      std::istringstream is{std::string(sv)};
      T v{}; is>>v;
      if(!is || (is>>std::ws && !is.eof())) return false;
      out=v; return true;
    }
  }

  template<class T>
  static std::string format(const T& v) {
    if constexpr (std::is_same_v<T,bool>) return v ? "true" : "false";
    else if constexpr (std::is_same_v<T,std::string>) return v;
    else { std::ostringstream os; os.precision(17); os<<v; return os.str(); }
  }

public:
  template<class T>
  void bind(std::string name, T& ref) {
    e_.push_back(Entry{
      std::move(name),
      [&ref](std::string_view s){ return parse<T>(s,ref); },
      [&ref](){ return format<T>(ref); }
    });
  }

  [[nodiscard]] bool set(std::string_view name, std::string_view value) {
    for(auto& x:e_) if(x.name==name) return x.set(value);
    return false;
  }

  [[nodiscard]] bool contains(std::string_view name) const {
    for(auto& x:e_) if(x.name==name) return true;
    return false;
  }

  // Returns false on unknown parameter or parse error.
  bool parse_argv(int argc, char** argv, int begin=1) {
    for(int i=begin;i<argc;++i){
      std::string s=argv[i];
      if(s.rfind("--",0)!=0) continue;
      s.erase(0,2);
      std::string key,val;
      auto eq=s.find('=');
      if(eq!=std::string::npos){ key=s.substr(0,eq); val=s.substr(eq+1); }
      else {
        key=s;
        if(i+1>=argc) return false;
        val=argv[++i];
      }
      if(!set(key,val)) return false;
    }
    return true;
  }

  // Simple format: key=value. Empty lines and lines beginning with # ignored.
  bool load_kv_file(const std::string& path) {
    std::ifstream f(path);
    if(!f) return false;
    std::string line;
    while(std::getline(f,line)){
      line=trim(line);
      if(line.empty()||line[0]=='#') continue;
      auto eq=line.find('=');
      if(eq==std::string::npos) return false;
      auto key=trim(line.substr(0,eq));
      auto val=trim(line.substr(eq+1));
      if(!set(key,val)) return false;
    }
    return true;
  }

  void dump(std::ostream& os, std::string_view prefix="[param] ") const {
    for(auto& x:e_) os<<prefix<<x.name<<'='<<x.get()<<'\n';
  }
};

} // namespace ahc

// ===== ahc/fixed_buffer.hpp =====
#line 1 "ahc/fixed_buffer.hpp"
#include <vector>
#include <cstddef>
#include <cassert>
#include <utility>
#include <span>

namespace ahc {

// Runtime-capacity vector that reserves once and never reallocates afterwards.
// push/emplace return false instead of growing beyond the configured capacity.
template<class T>
class FixedBuffer {
  std::vector<T> v_;
  size_t cap_ = 0;
public:
  FixedBuffer() = default;
  explicit FixedBuffer(size_t capacity) { reset_capacity(capacity); }

  void reset_capacity(size_t capacity) {
    cap_ = capacity;
    v_.clear();
    v_.shrink_to_fit();
    v_.reserve(cap_);
  }
  void clear() noexcept { v_.clear(); }
  [[nodiscard]] size_t size() const noexcept { return v_.size(); }
  [[nodiscard]] size_t capacity() const noexcept { return cap_; }
  [[nodiscard]] bool empty() const noexcept { return v_.empty(); }
  [[nodiscard]] bool full() const noexcept { return v_.size() >= cap_; }

  bool push_back(const T& x) {
    if (full()) return false;
    v_.push_back(x);
    return true;
  }
  bool push_back(T&& x) {
    if (full()) return false;
    v_.push_back(std::move(x));
    return true;
  }
  template<class... Args>
  bool emplace_back(Args&&... args) {
    if (full()) return false;
    v_.emplace_back(std::forward<Args>(args)...);
    return true;
  }

  T& operator[](size_t i) { return v_[i]; }
  const T& operator[](size_t i) const { return v_[i]; }
  T& back() { return v_.back(); }
  const T& back() const { return v_.back(); }
  auto begin() noexcept { return v_.begin(); }
  auto end() noexcept { return v_.end(); }
  auto begin() const noexcept { return v_.begin(); }
  auto end() const noexcept { return v_.end(); }
  std::span<T> span() noexcept { return {v_.data(), v_.size()}; }
  std::span<const T> span() const noexcept { return {v_.data(), v_.size()}; }
  std::vector<T>& vector() noexcept { return v_; }
  const std::vector<T>& vector() const noexcept { return v_; }
};

} // namespace ahc

// ===== ahc/eval_cache.hpp =====
#line 1 "ahc/eval_cache.hpp"
#include <vector>
#include <cstdint>
#include <limits>
#include <utility>

namespace ahc {

// Tiny direct-mapped cache for expensive evaluations keyed by a 64-bit state
// hash. It deliberately trades exact retention for fixed memory and very small
// constant factors. clear() is O(1) through generation stamps.
template<class Value>
class EvalCache64 {
  struct Entry {
    uint64_t key = 0;
    Value value{};
    uint32_t stamp = 0;
  };
  std::vector<Entry> tab_;
  size_t mask_ = 0;
  uint32_t stamp_ = 1;
  uint64_t hits_ = 0, misses_ = 0;

  static size_t ceil_pow2(size_t x) {
    size_t n = 1;
    while (n < x) n <<= 1;
    return n;
  }
  size_t slot(uint64_t key) const {
    return static_cast<size_t>(mix64(key)) & mask_;
  }
public:
  EvalCache64() { reset(1024); }
  explicit EvalCache64(size_t capacity) { reset(capacity); }

  void reset(size_t capacity) {
    size_t n = ceil_pow2(std::max<size_t>(1, capacity));
    tab_.assign(n, Entry{});
    mask_ = n - 1;
    stamp_ = 1;
    hits_ = misses_ = 0;
  }
  void clear() {
    ++stamp_;
    if (stamp_ == 0) {
      for (auto& e : tab_) e.stamp = 0;
      stamp_ = 1;
    }
  }
  void clear_stats() { hits_ = misses_ = 0; }
  [[nodiscard]] uint64_t hits() const { return hits_; }
  [[nodiscard]] uint64_t misses() const { return misses_; }
  [[nodiscard]] double hit_rate() const {
    uint64_t n = hits_ + misses_;
    return n ? double(hits_) / double(n) : 0.0;
  }

  bool get(uint64_t key, Value& out) {
    auto& e = tab_[slot(key)];
    if (e.stamp == stamp_ && e.key == key) {
      out = e.value;
      ++hits_;
      return true;
    }
    ++misses_;
    return false;
  }
  const Value* find(uint64_t key) const {
    const auto& e = tab_[slot(key)];
    return (e.stamp == stamp_ && e.key == key) ? &e.value : nullptr;
  }
  void put(uint64_t key, const Value& value) {
    auto& e = tab_[slot(key)];
    e.key = key;
    e.value = value;
    e.stamp = stamp_;
  }
  void put(uint64_t key, Value&& value) {
    auto& e = tab_[slot(key)];
    e.key = key;
    e.value = std::move(value);
    e.stamp = stamp_;
  }
};

} // namespace ahc

// ===== ahc/cost_aware_operator.hpp =====
#line 1 "ahc/cost_aware_operator.hpp"
#include <vector>
#include <cmath>
#include <algorithm>
#include <cassert>

namespace ahc {

// Cost-aware UCB selector for neighborhoods. Unlike AdaptiveOperatorSelector,
// credit is normalized by measured wall time, so an expensive destroy/repair
// competes fairly with very cheap local moves.
class CostAwareOperatorSelector {
public:
  struct Config {
    double exploration = 0.15;
    double ema_alpha = 0.12;
    double accepted_bonus = 0.0;
    double best_bonus = 0.02;
    double min_seconds = 1e-7;
  };
  struct Stats {
    int pulls = 0;
    int accepted = 0;
    int best_updates = 0;
    double seconds = 0.0;
    double total_gain = 0.0;
    double ema_gain_rate = 0.0;
    bool enabled = true;
  };
private:
  std::vector<Stats> a_;
  Config cfg_{};
  int total_pulls_ = 0;
public:
  CostAwareOperatorSelector() = default;
  explicit CostAwareOperatorSelector(int n) : a_(n) {}
  CostAwareOperatorSelector(int n, Config cfg) : a_(n), cfg_(cfg) {}

  void reset(int n) { a_.assign(n, Stats{}); total_pulls_ = 0; }
  [[nodiscard]] int size() const { return static_cast<int>(a_.size()); }
  void set_enabled(int i, bool enabled) { assert(0 <= i && i < size()); a_[i].enabled = enabled; }
  [[nodiscard]] const Stats& stats(int i) const { assert(0 <= i && i < size()); return a_[i]; }

  int choose() const {
    for (int i = 0; i < size(); ++i) if (a_[i].enabled && a_[i].pulls == 0) return i;
    int best = -1;
    double best_value = -1e300;
    const double logn = std::log(2.0 + total_pulls_);
    for (int i = 0; i < size(); ++i) {
      if (!a_[i].enabled) continue;
      const auto& s = a_[i];
      double ucb = s.ema_gain_rate + cfg_.exploration * std::sqrt(logn / std::max(1, s.pulls));
      if (ucb > best_value) { best_value = ucb; best = i; }
    }
    return best;
  }

  // gain should be nonnegative and preferably normalized, e.g.
  // max(0, old_score-new_score)/max(1, abs(old_score)) for minimization.
  void observe(int i, double seconds, double gain, bool accepted, bool new_best) {
    assert(0 <= i && i < size());
    auto& s = a_[i];
    seconds = std::max(seconds, cfg_.min_seconds);
    double reward = std::max(0.0, gain)
                  + (accepted ? cfg_.accepted_bonus : 0.0)
                  + (new_best ? cfg_.best_bonus : 0.0);
    double rate = reward / seconds;
    if (s.pulls == 0) s.ema_gain_rate = rate;
    else s.ema_gain_rate = (1.0 - cfg_.ema_alpha) * s.ema_gain_rate + cfg_.ema_alpha * rate;
    ++s.pulls;
    ++total_pulls_;
    s.accepted += accepted;
    s.best_updates += new_best;
    s.seconds += seconds;
    s.total_gain += gain;
  }
};

} // namespace ahc

// ===== ahc/small_exact.hpp =====
#line 1 "ahc/small_exact.hpp"
#include <vector>
#include <cstddef>
#include <limits>
#include <type_traits>
#include <cassert>
#include <algorithm>

namespace ahc {

template<class T>
class SmallSubsetTable {
  int n_ = 0;
  std::vector<T> a_;
public:
  SmallSubsetTable() = default;
  SmallSubsetTable(int n, const T& init = T{}) { reset(n, init); }
  void reset(int n, const T& init = T{}) {
    assert(0 <= n && n <= 25);
    n_ = n;
    a_.assign(size_t(1) << n_, init);
  }
  [[nodiscard]] int bits() const { return n_; }
  [[nodiscard]] size_t size() const { return a_.size(); }
  T& operator[](size_t mask) { return a_[mask]; }
  const T& operator[](size_t mask) const { return a_[mask]; }
  void fill(const T& x) { std::fill(a_.begin(), a_.end(), x); }
};

template<class F>
inline void for_each_submask(uint64_t mask, F&& f) {
  uint64_t s = mask;
  while (true) {
    f(s);
    if (s == 0) break;
    s = (s - 1) & mask;
  }
}

// Lightweight minimization branch-and-bound accounting helper. It centralizes
// incumbent updates, bound pruning, node limits and a child time budget without
// dictating the user's DFS representation.
template<class Score>
class BranchAndBoundController {
  Score incumbent_{};
  bool has_incumbent_ = false;
  size_t node_limit_ = std::numeric_limits<size_t>::max();
  size_t nodes_ = 0, pruned_ = 0, improved_ = 0;
  TimeSlice slice_;
public:
  BranchAndBoundController(const Timer& timer, double seconds,
                           size_t node_limit = std::numeric_limits<size_t>::max())
      : node_limit_(node_limit), slice_(timer, seconds) {}

  void set_incumbent(const Score& s) { incumbent_ = s; has_incumbent_ = true; }
  [[nodiscard]] bool has_incumbent() const { return has_incumbent_; }
  [[nodiscard]] const Score& incumbent() const { return incumbent_; }
  [[nodiscard]] size_t nodes() const { return nodes_; }
  [[nodiscard]] size_t pruned() const { return pruned_; }
  [[nodiscard]] size_t improvements() const { return improved_; }
  [[nodiscard]] bool exhausted() const { return nodes_ >= node_limit_ || slice_.expired(); }

  // Call once when entering a node. lower_bound is a valid optimistic bound.
  bool enter(const Score& lower_bound) {
    if (exhausted()) return false;
    ++nodes_;
    if (has_incumbent_ && !(lower_bound < incumbent_)) {
      ++pruned_;
      return false;
    }
    return true;
  }
  bool improve(const Score& exact_score) {
    if (!has_incumbent_ || exact_score < incumbent_) {
      incumbent_ = exact_score;
      has_incumbent_ = true;
      ++improved_;
      return true;
    }
    return false;
  }
};

} // namespace ahc

// ===== ahc/zero_one_bfs.hpp =====
#line 1 "ahc/zero_one_bfs.hpp"
#include <vector>
#include <deque>
#include <limits>
#include <cassert>

namespace ahc {

// Reusable 0-1 BFS workspace. expand(u, relax) must call relax(v, w) with
// w in {0,1}. Only touched vertices are reset between runs.
class ZeroOneBfsWorkspace {
  static constexpr int INF = std::numeric_limits<int>::max() / 4;
  std::vector<int> dist_;
  std::vector<int> touched_;
  std::vector<unsigned char> touched_flag_;
  std::deque<int> q_;
public:
  ZeroOneBfsWorkspace() = default;
  explicit ZeroOneBfsWorkspace(int n) { reset_size(n); }
  void reset_size(int n) {
    dist_.assign(n, INF);
    touched_flag_.assign(n, 0);
    touched_.clear();
    q_.clear();
  }
  [[nodiscard]] int size() const { return static_cast<int>(dist_.size()); }
  [[nodiscard]] int distance(int v) const { return dist_[v]; }
  [[nodiscard]] bool reachable(int v) const { return dist_[v] < INF; }

  template<class Expand>
  void run(int source, Expand&& expand) {
    assert(0 <= source && source < size());
    for (int v : touched_) { dist_[v] = INF; touched_flag_[v] = 0; }
    touched_.clear();
    q_.clear();
    auto touch = [&](int v) {
      if (!touched_flag_[v]) { touched_flag_[v] = 1; touched_.push_back(v); }
    };
    dist_[source] = 0;
    touch(source);
    q_.push_back(source);
    while (!q_.empty()) {
      int u = q_.front(); q_.pop_front();
      int du = dist_[u];
      expand(u, [&](int v, int w) {
        assert(0 <= v && v < size());
        assert(w == 0 || w == 1);
        int nd = du + w;
        if (nd >= dist_[v]) return;
        dist_[v] = nd;
        touch(v);
        if (w == 0) q_.push_front(v);
        else q_.push_back(v);
      });
    }
  }
};

} // namespace ahc

// ===== ahc/grid_distance_cache.hpp =====
#line 1 "ahc/grid_distance_cache.hpp"
#include <vector>
#include <cstdint>
#include <limits>
#include <cassert>
#include <queue>
#include <algorithm>

namespace ahc {

// Lazy BFS-distance rows for a runtime HxW 4-neighbor grid. Suitable for AHC
// boards around <= 400 cells where a handful of repeatedly queried sources
// dominate. Rows are cached with simple round-robin replacement.
class GridDistanceCache {
public:
  static constexpr uint16_t INF = std::numeric_limits<uint16_t>::max();
private:
  int h_ = 0, w_ = 0, n_ = 0, cap_ = 0, next_slot_ = 0;
  std::vector<unsigned char> passable_;
  std::vector<int> source_to_slot_, slot_source_;
  std::vector<std::vector<uint16_t>> rows_;
  std::vector<int> q_;

  void build_row(int src, std::vector<uint16_t>& d) {
    d.assign(n_, INF);
    q_.clear();
    q_.reserve(n_);
    d[src] = 0;
    q_.push_back(src);
    for (size_t qi = 0; qi < q_.size(); ++qi) {
      int u = q_[qi];
      int y = u / w_, x = u % w_;
      const int dy[4] = {-1, 1, 0, 0};
      const int dx[4] = {0, 0, -1, 1};
      for (int k = 0; k < 4; ++k) {
        int ny = y + dy[k], nx = x + dx[k];
        if (ny < 0 || ny >= h_ || nx < 0 || nx >= w_) continue;
        int v = ny * w_ + nx;
        if (!passable_[v] || d[v] != INF) continue;
        d[v] = static_cast<uint16_t>(d[u] + 1);
        q_.push_back(v);
      }
    }
  }
public:
  GridDistanceCache() = default;
  GridDistanceCache(int h, int w, std::vector<unsigned char> passable, int cache_rows = 16) {
    reset(h, w, std::move(passable), cache_rows);
  }
  void reset(int h, int w, std::vector<unsigned char> passable, int cache_rows = 16) {
    h_ = h; w_ = w; n_ = h_ * w_; cap_ = std::max(1, cache_rows); next_slot_ = 0;
    assert(static_cast<int>(passable.size()) == n_);
    passable_ = std::move(passable);
    source_to_slot_.assign(n_, -1);
    slot_source_.assign(cap_, -1);
    rows_.assign(cap_, {});
    q_.reserve(n_);
  }
  [[nodiscard]] int height() const { return h_; }
  [[nodiscard]] int width() const { return w_; }
  [[nodiscard]] bool passable(int v) const { return passable_[v] != 0; }

  const std::vector<uint16_t>& row(int src) {
    assert(0 <= src && src < n_ && passable_[src]);
    int slot = source_to_slot_[src];
    if (slot >= 0) return rows_[slot];
    slot = next_slot_++ % cap_;
    int old = slot_source_[slot];
    if (old >= 0) source_to_slot_[old] = -1;
    build_row(src, rows_[slot]);
    source_to_slot_[src] = slot;
    slot_source_[slot] = src;
    return rows_[slot];
  }
  uint16_t distance(int src, int dst) { return row(src)[dst]; }
  void clear() {
    std::fill(source_to_slot_.begin(), source_to_slot_.end(), -1);
    std::fill(slot_source_.begin(), slot_source_.end(), -1);
    next_slot_ = 0;
  }
};

} // namespace ahc

// ===== ahc/debug.hpp =====
#line 1 "ahc/debug.hpp"
#include <iostream>
#include <cstdlib>

#ifndef AHC_DLOG
# ifdef LOCAL
#  define AHC_DLOG(expr) do { std::cerr << expr << '\n'; } while (0)
#  define AHC_DLOG_EVERY(iter, mask, expr) do { if ((((unsigned long long)(iter)) & (unsigned long long)(mask)) == 0ULL) { std::cerr << expr << '\n'; } } while (0)
#  define AHC_CHECK(cond) do { if (!(cond)) { std::cerr << "AHC_CHECK failed: " #cond " @ " << __FILE__ << ':' << __LINE__ << '\n'; std::abort(); } } while (0)
# else
#  define AHC_DLOG(expr) do {} while (0)
#  define AHC_DLOG_EVERY(iter, mask, expr) do {} while (0)
#  define AHC_CHECK(cond) do { (void)sizeof(cond); } while (0)
# endif
#endif

// ===== ahc/bounded_best_first.hpp =====
#line 1 "ahc/bounded_best_first.hpp"
#include <vector>
#include <queue>
#include <cstdint>
#include <cassert>
#include <functional>
#include <utility>

namespace ahc {

// Online best-first frontier with a hard live-node cap. When full, the current
// worst live entry is discarded. Two lazy heaps provide best-pop and worst-prune
// while slots are recycled, avoiding unbounded Node storage.
template<class T, class Priority, class Better = std::less<Priority>>
class BoundedBestFirst {
  struct Slot {
    T value{};
    Priority pri{};
    uint32_t gen = 0;
    bool active = false;
  };
  struct Item {
    Priority pri{};
    int id = -1;
    uint32_t gen = 0;
  };
  struct BestCmp {
    Better better{};
    bool operator()(const Item& a, const Item& b) const {
      if (better(a.pri, b.pri)) return false;
      if (better(b.pri, a.pri)) return true;
      return a.id > b.id;
    }
  };
  struct WorstCmp {
    Better better{};
    bool operator()(const Item& a, const Item& b) const {
      if (better(a.pri, b.pri)) return true;
      if (better(b.pri, a.pri)) return false;
      return a.id > b.id;
    }
  };

  size_t cap_ = 0, live_ = 0;
  Better better_{};
  std::vector<Slot> slots_;
  std::vector<int> free_;
  std::priority_queue<Item, std::vector<Item>, BestCmp> best_;
  std::priority_queue<Item, std::vector<Item>, WorstCmp> worst_;

  bool valid(const Item& x) const {
    return x.id >= 0 && x.id < static_cast<int>(slots_.size()) &&
           slots_[x.id].active && slots_[x.id].gen == x.gen;
  }
  void clean_best() { while (!best_.empty() && !valid(best_.top())) best_.pop(); }
  void clean_worst() { while (!worst_.empty() && !valid(worst_.top())) worst_.pop(); }

  int allocate_slot() {
    if (!free_.empty()) {
      int id = free_.back(); free_.pop_back();
      return id;
    }
    slots_.push_back(Slot{});
    return static_cast<int>(slots_.size()) - 1;
  }
  void deactivate(int id) {
    auto& s = slots_[id];
    if (!s.active) return;
    s.active = false;
    --live_;
    free_.push_back(id);
  }
  void maybe_rebuild() {
    const size_t lim = std::max<size_t>(1024, 4 * std::max<size_t>(1, cap_));
    if (best_.size() <= lim && worst_.size() <= lim) return;
    decltype(best_) nb{BestCmp{better_}};
    decltype(worst_) nw{WorstCmp{better_}};
    for (int id = 0; id < static_cast<int>(slots_.size()); ++id) if (slots_[id].active) {
      Item x{slots_[id].pri, id, slots_[id].gen};
      nb.push(x); nw.push(x);
    }
    best_.swap(nb); worst_.swap(nw);
  }

public:
  BoundedBestFirst() = default;
  explicit BoundedBestFirst(size_t capacity, Better better = Better{})
      : cap_(capacity), better_(better), best_(BestCmp{better}), worst_(WorstCmp{better}) {
    slots_.reserve(capacity);
    free_.reserve(capacity);
  }

  void reset(size_t capacity) {
    cap_ = capacity; live_ = 0;
    slots_.clear(); free_.clear();
    best_ = decltype(best_)(BestCmp{better_});
    worst_ = decltype(worst_)(WorstCmp{better_});
    slots_.reserve(capacity); free_.reserve(capacity);
  }
  void clear() { reset(cap_); }
  [[nodiscard]] size_t size() const { return live_; }
  [[nodiscard]] bool empty() const { return live_ == 0; }
  [[nodiscard]] size_t capacity() const { return cap_; }

  // Returns true iff the inserted item survives the capacity pruning.
  bool push(const T& value, const Priority& pri) {
    if (cap_ == 0) return false;
    int id = allocate_slot();
    auto& s = slots_[id];
    ++s.gen; if (s.gen == 0) ++s.gen;
    s.value = value; s.pri = pri; s.active = true; ++live_;
    Item x{pri, id, s.gen};
    best_.push(x); worst_.push(x);
    bool survived = true;
    if (live_ > cap_) {
      clean_worst();
      int kill = worst_.top().id;
      worst_.pop();
      survived = kill != id;
      deactivate(kill);
    }
    maybe_rebuild();
    return survived;
  }
  bool push(T&& value, const Priority& pri) {
    if (cap_ == 0) return false;
    int id = allocate_slot();
    auto& s = slots_[id];
    ++s.gen; if (s.gen == 0) ++s.gen;
    s.value = std::move(value); s.pri = pri; s.active = true; ++live_;
    Item x{pri, id, s.gen};
    best_.push(x); worst_.push(x);
    bool survived = true;
    if (live_ > cap_) {
      clean_worst();
      int kill = worst_.top().id;
      worst_.pop();
      survived = kill != id;
      deactivate(kill);
    }
    maybe_rebuild();
    return survived;
  }

  // Returns {value, priority} of the current best entry and removes it.
  std::pair<T, Priority> pop_best() {
    assert(live_ > 0);
    clean_best();
    Item x = best_.top(); best_.pop();
    auto& s = slots_[x.id];
    std::pair<T, Priority> out{std::move(s.value), s.pri};
    deactivate(x.id);
    return out;
  }
};


// ===== v0.9: acceptance policies / sampling / tabu / replay =====

template<class Score, class Better = std::less<Score>>
class LateAcceptanceController {
  std::vector<Score> history_;
  size_t pos_ = 0;
  Better better_{};
  bool initialized_ = false;
public:
  LateAcceptanceController() = default;
  explicit LateAcceptanceController(size_t window, Better better = Better{})
      : history_(std::max<size_t>(1, window)), better_(better) {}

  void reset(Score initial) {
    if (history_.empty()) history_.resize(1);
    std::fill(history_.begin(), history_.end(), initial);
    pos_ = 0;
    initialized_ = true;
  }
  [[nodiscard]] size_t window() const { return history_.size(); }

  // Default Better=less means minimization. The current score is updated on acceptance.
  bool accept(Score candidate, Score& current) {
    assert(initialized_);
    const Score old = history_[pos_];
    const bool non_worse_current = !better_(current, candidate); // candidate <= current
    const bool beats_late = better_(candidate, old) || !better_(old, candidate); // candidate <= old
    const bool ok = non_worse_current || beats_late;
    if (ok) current = candidate;
    history_[pos_] = current;
    pos_ = (pos_ + 1) % history_.size();
    return ok;
  }
};

class ThresholdAcceptanceController {
  double start_ = 0.0, end_ = 0.0, exponent_ = 1.0;
public:
  ThresholdAcceptanceController() = default;
  ThresholdAcceptanceController(double start_threshold, double end_threshold,
                                double exponent = 1.0)
      : start_(std::max(0.0, start_threshold)),
        end_(std::max(0.0, end_threshold)),
        exponent_(std::max(1e-9, exponent)) {}

  [[nodiscard]] double threshold(double progress) const {
    double p = std::pow(std::clamp(progress, 0.0, 1.0), exponent_);
    return start_ + (end_ - start_) * p;
  }
  // Minimization convention: delta = candidate - current.
  [[nodiscard]] bool accept(double delta, double progress) const {
    return delta <= threshold(progress);
  }
};

class RecordToRecordController {
  double start_dev_ = 0.0, end_dev_ = 0.0, exponent_ = 1.0;
  double best_ = std::numeric_limits<double>::infinity();
public:
  RecordToRecordController() = default;
  RecordToRecordController(double start_deviation, double end_deviation,
                           double exponent = 1.0)
      : start_dev_(std::max(0.0, start_deviation)),
        end_dev_(std::max(0.0, end_deviation)),
        exponent_(std::max(1e-9, exponent)) {}

  void reset(double initial) { best_ = initial; }
  [[nodiscard]] double best() const { return best_; }
  [[nodiscard]] double deviation(double progress) const {
    double p = std::pow(std::clamp(progress, 0.0, 1.0), exponent_);
    return start_dev_ + (end_dev_ - start_dev_) * p;
  }
  // Minimization, absolute score deviation from the best record.
  bool accept(double candidate, double progress) {
    if (candidate < best_) best_ = candidate;
    return candidate <= best_ + deviation(progress);
  }
};

// Fixed-memory approximate tabu table. Hash collisions only forget old tabu entries;
// they never create a long-lived unrelated entry because the full 64-bit key is checked.
class TabuCache64 {
  struct Entry { uint64_t key = 0; uint64_t until = 0; uint32_t stamp = 0; };
  std::vector<Entry> tab_;
  size_t mask_ = 0;
  uint32_t epoch_ = 1;
  static size_t ceil_pow2(size_t n) {
    size_t x = 1;
    while (x < std::max<size_t>(2, n)) x <<= 1;
    return x;
  }
public:
  TabuCache64() = default;
  explicit TabuCache64(size_t capacity) { reset(capacity); }
  void reset(size_t capacity) {
    size_t n = ceil_pow2(capacity);
    tab_.assign(n, Entry{}); mask_ = n - 1; epoch_ = 1;
  }
  void clear() {
    if (++epoch_ == 0) { for (auto& e : tab_) e.stamp = 0; epoch_ = 1; }
  }
  [[nodiscard]] bool contains(uint64_t key, uint64_t iteration) const {
    if (tab_.empty()) return false;
    const auto& e = tab_[mix64(key) & mask_];
    return e.stamp == epoch_ && e.key == key && iteration < e.until;
  }
  void insert(uint64_t key, uint64_t iteration, uint64_t tenure) {
    if (tab_.empty()) return;
    auto& e = tab_[mix64(key) & mask_];
    e.key = key; e.until = iteration + tenure; e.stamp = epoch_;
  }
  [[nodiscard]] bool allowed(uint64_t key, uint64_t iteration, bool aspiration = false) const {
    return aspiration || !contains(key, iteration);
  }
};

template<class T>
class ReservoirSampler {
  std::vector<T> sample_;
  size_t k_ = 0;
  uint64_t seen_ = 0;
public:
  ReservoirSampler() = default;
  explicit ReservoirSampler(size_t k) { reset(k); }
  void reset(size_t k) { k_ = k; sample_.clear(); sample_.reserve(k); seen_ = 0; }
  void clear() { sample_.clear(); seen_ = 0; }
  [[nodiscard]] size_t capacity() const { return k_; }
  [[nodiscard]] uint64_t seen() const { return seen_; }
  [[nodiscard]] const std::vector<T>& values() const { return sample_; }

  void consider(const T& x, Rng& rng) {
    ++seen_;
    const size_t k = k_;
    if (k == 0) return;
    if (sample_.size() < k) { sample_.push_back(x); return; }
    uint64_t j = rng.u64() % seen_;
    if (j < k) sample_[static_cast<size_t>(j)] = x;
  }
  void consider(T&& x, Rng& rng) {
    ++seen_;
    const size_t k = k_;
    if (k == 0) return;
    if (sample_.size() < k) { sample_.push_back(std::move(x)); return; }
    uint64_t j = rng.u64() % seen_;
    if (j < k) sample_[static_cast<size_t>(j)] = std::move(x);
  }
};

// Per-index lazy value cache with O(1) full invalidation and O(1) point invalidation.
template<class T>
class EpochValueCache {
  std::vector<T> value_;
  std::vector<uint32_t> stamp_;
  uint32_t epoch_ = 1;
public:
  EpochValueCache() = default;
  explicit EpochValueCache(size_t n) { reset(n); }
  void reset(size_t n) { value_.resize(n); stamp_.assign(n, 0); epoch_ = 1; }
  [[nodiscard]] size_t size() const { return value_.size(); }
  void clear() {
    if (++epoch_ == 0) { std::fill(stamp_.begin(), stamp_.end(), 0); epoch_ = 1; }
  }
  void invalidate(size_t i) { assert(i < size()); stamp_[i] = 0; }
  [[nodiscard]] bool valid(size_t i) const { return i < size() && stamp_[i] == epoch_; }
  const T& get(size_t i) const { assert(valid(i)); return value_[i]; }
  void set(size_t i, const T& v) { assert(i < size()); value_[i] = v; stamp_[i] = epoch_; }
  template<class F>
  const T& get_or_compute(size_t i, F&& f) {
    assert(i < size());
    if (stamp_[i] != epoch_) { value_[i] = std::forward<F>(f)(); stamp_[i] = epoch_; }
    return value_[i];
  }
};

// A cheap rolling digest for deterministic replay/regression checks.
class TraceDigest {
  uint64_t h_ = 0x243f6a8885a308d3ULL;
  uint64_t n_ = 0;
public:
  void reset(uint64_t seed = 0x243f6a8885a308d3ULL) { h_ = seed; n_ = 0; }
  void add_u64(uint64_t x) {
    h_ = mix64(h_ ^ mix64(x + 0x9e3779b97f4a7c15ULL + n_)); ++n_;
  }
  template<class Int> requires std::is_integral_v<Int>
  void add(Int x) { add_u64(static_cast<uint64_t>(x)); }
  void add_double(double x) {
    uint64_t bits = 0; static_assert(sizeof(bits) == sizeof(x));
    std::memcpy(&bits, &x, sizeof(bits)); add_u64(bits);
  }
  [[nodiscard]] uint64_t value() const { return h_; }
  [[nodiscard]] uint64_t count() const { return n_; }
};

// Convenience wrapper around CostAwareOperatorSelector that keeps human-readable
// neighborhood names and exposes a compact report for LOCAL experiments.
class NamedNeighborhoodPortfolio {
  std::vector<std::string> names_;
  CostAwareOperatorSelector selector_;
public:
  NamedNeighborhoodPortfolio() = default;
  explicit NamedNeighborhoodPortfolio(std::vector<std::string> names)
      : names_(std::move(names)), selector_(static_cast<int>(names_.size())) {
    if (names_.empty()) { names_.push_back("op0"); selector_ = CostAwareOperatorSelector(1); }
  }
  [[nodiscard]] int choose() { return selector_.choose(); }
  void observe(int op, double seconds, double normalized_gain,
               bool accepted, bool best_update) {
    selector_.observe(op, seconds, normalized_gain, accepted, best_update);
  }
  [[nodiscard]] const std::string& name(int op) const { return names_.at(op); }
  void report(std::ostream& os) const {
    for (int i = 0; i < selector_.size(); ++i) {
      const auto& st = selector_.stats(i);
      os << names_[i]
         << " pulls=" << st.pulls
         << " accepted=" << st.accepted
         << " best=" << st.best_updates
         << " sec=" << st.seconds
         << " gain=" << st.total_gain
         << " gain_rate_ema=" << st.ema_gain_rate
         << '\n';
    }
  }
};

} // namespace ahc

// ===== v1.1: repair / weighted sampling / spatial buckets / scalar undo =====
#include <vector>
#include <array>
#include <cstddef>
#include <cstdint>
#include <cmath>
#include <limits>
#include <algorithm>
#include <cassert>
#include <type_traits>
#include <cstring>
#include <utility>

namespace ahc {

// Dynamic non-negative weighted sampling over indices.
// set/add: O(log n), sample: O(log n), total: O(1).
class WeightedIndexSampler {
  std::vector<double> bit_;
  std::vector<double> weight_;
  double total_ = 0.0;

  void bit_add(size_t i, double delta) {
    for (size_t x = i + 1; x < bit_.size(); x += x & -x) bit_[x] += delta;
  }
public:
  WeightedIndexSampler() = default;
  explicit WeightedIndexSampler(size_t n) { reset(n); }

  void reset(size_t n) {
    weight_.assign(n, 0.0);
    bit_.assign(n + 1, 0.0);
    total_ = 0.0;
  }
  [[nodiscard]] size_t size() const { return weight_.size(); }
  [[nodiscard]] double total() const { return total_; }
  [[nodiscard]] double weight(size_t i) const { assert(i < size()); return weight_[i]; }

  void set(size_t i, double w) {
    assert(i < size());
    if (!(w >= 0.0) || !std::isfinite(w)) w = 0.0;
    double d = w - weight_[i];
    weight_[i] = w;
    total_ += d;
    bit_add(i, d);
    if (std::abs(total_) < 1e-15) total_ = 0.0;
  }
  void add(size_t i, double delta) { set(i, std::max(0.0, weight(i) + delta)); }
  void clear_weights() {
    std::fill(weight_.begin(), weight_.end(), 0.0);
    std::fill(bit_.begin(), bit_.end(), 0.0);
    total_ = 0.0;
  }

  // Returns -1 if all weights are zero.
  [[nodiscard]] int sample(Rng& rng) const {
    if (!(total_ > 0.0) || weight_.empty()) return -1;
    double target = rng.uniform01() * total_;
    size_t idx = 0;
    double acc = 0.0;
    size_t step = 1;
    while ((step << 1) <= weight_.size()) step <<= 1;
    for (; step; step >>= 1) {
      size_t nxt = idx + step;
      if (nxt <= weight_.size() && acc + bit_[nxt] <= target) {
        idx = nxt;
        acc += bit_[nxt];
      }
    }
    if (idx >= weight_.size()) idx = weight_.size() - 1;
    return static_cast<int>(idx);
  }
};

// O(1) sampling for a static weight vector. Rebuild is O(n).
class AliasSampler {
  std::vector<double> prob_;
  std::vector<int> alias_;
  bool valid_ = false;
public:
  AliasSampler() = default;
  explicit AliasSampler(const std::vector<double>& weights) { build(weights); }

  void build(const std::vector<double>& weights) {
    const int n = static_cast<int>(weights.size());
    prob_.assign(n, 0.0);
    alias_.resize(n);
    valid_ = false;
    if (n == 0) return;
    double maxw = 0.0;
    for (double w : weights) if (w > 0.0 && std::isfinite(w)) maxw = std::max(maxw, w);
    if (!(maxw > 0.0)) return;
    double norm_sum = 0.0;
    for (double w : weights) if (w > 0.0 && std::isfinite(w)) norm_sum += w / maxw;
    if (!(norm_sum > 0.0) || !std::isfinite(norm_sum)) return;

    std::vector<double> scaled(n);
    std::vector<int> small, large;
    small.reserve(n); large.reserve(n);
    for (int i = 0; i < n; ++i) {
      double w = (weights[i] > 0.0 && std::isfinite(weights[i])) ? weights[i] / maxw : 0.0;
      scaled[i] = w * n / norm_sum;
      (scaled[i] < 1.0 ? small : large).push_back(i);
      alias_[i] = i;
    }
    while (!small.empty() && !large.empty()) {
      int s = small.back(); small.pop_back();
      int l = large.back(); large.pop_back();
      prob_[s] = scaled[s];
      alias_[s] = l;
      scaled[l] = (scaled[l] + scaled[s]) - 1.0;
      (scaled[l] < 1.0 ? small : large).push_back(l);
    }
    for (int i : large) prob_[i] = 1.0;
    for (int i : small) prob_[i] = 1.0;
    valid_ = true;
  }
  [[nodiscard]] int size() const { return static_cast<int>(prob_.size()); }
  [[nodiscard]] bool valid() const { return valid_; }
  [[nodiscard]] int sample(Rng& rng) const {
    if (!valid_ || prob_.empty()) return -1;
    int i = rng.int_range(0, static_cast<int>(prob_.size()));
    return rng.uniform01() < prob_[i] ? i : alias_[i];
  }
};

// Stable soft-min stochastic choice. Useful for randomized greedy construction.
// Scores are minimized. temp<=0 chooses a deterministic argmin.
template<class Score>
int softmin_sample(const std::vector<Score>& scores, double temp, Rng& rng) {
  if (scores.empty()) return -1;
  int best = 0;
  for (int i = 1; i < static_cast<int>(scores.size()); ++i)
    if (scores[i] < scores[best]) best = i;
  if (!(temp > 0.0)) return best;
  const double base = static_cast<double>(scores[best]);
  std::vector<double> w(scores.size());
  double total = 0.0;
  for (int i = 0; i < static_cast<int>(scores.size()); ++i) {
    double z = -(static_cast<double>(scores[i]) - base) / temp;
    double wi = z < -60.0 ? 0.0 : std::exp(z);
    w[i] = wi; total += wi;
  }
  if (!(total > 0.0)) return best;
  double x = rng.uniform01() * total;
  for (int i = 0; i < static_cast<int>(w.size()); ++i) {
    if (x < w[i]) return i;
    x -= w[i];
  }
  return static_cast<int>(w.size()) - 1;
}

// Multiplicative penalty controller targeting a desired fraction of infeasible moves.
// Call observe(violation) once per evaluated/accepted state according to your policy.
class AdaptivePenaltyController {
public:
  struct Config {
    double initial = 1.0;
    double min_penalty = 1e-9;
    double max_penalty = 1e18;
    double target_violation_rate = 0.20;
    double gain = 1.5;
    int window = 128;
  };
private:
  Config cfg_{};
  double penalty_ = 1.0;
  int seen_ = 0, violated_ = 0;
  double violation_sum_ = 0.0;
public:
  AdaptivePenaltyController() { reset(Config{}); }
  explicit AdaptivePenaltyController(Config cfg) { reset(cfg); }
  void reset(Config cfg) {
    cfg_ = cfg;
    cfg_.window = std::max(1, cfg_.window);
    cfg_.target_violation_rate = std::clamp(cfg_.target_violation_rate, 0.0, 1.0);
    cfg_.min_penalty = std::max(0.0, cfg_.min_penalty);
    cfg_.max_penalty = std::max(cfg_.min_penalty, cfg_.max_penalty);
    penalty_ = std::clamp(cfg_.initial, cfg_.min_penalty, cfg_.max_penalty);
    seen_ = violated_ = 0; violation_sum_ = 0.0;
  }
  [[nodiscard]] double penalty() const { return penalty_; }
  [[nodiscard]] int pending() const { return seen_; }

  // violation<=0 means feasible. Returns true when the penalty was updated.
  bool observe(double violation) {
    ++seen_;
    if (violation > 0.0) { ++violated_; violation_sum_ += violation; }
    if (seen_ < cfg_.window) return false;
    double rate = static_cast<double>(violated_) / seen_;
    // Exponential update is scale-free and symmetric in log space.
    penalty_ *= std::exp(cfg_.gain * (rate - cfg_.target_violation_rate));
    penalty_ = std::clamp(penalty_, cfg_.min_penalty, cfg_.max_penalty);
    seen_ = violated_ = 0; violation_sum_ = 0.0;
    return true;
  }
  [[nodiscard]] double penalize(double objective, double violation) const {
    return objective + penalty_ * std::max(0.0, violation);
  }
};

// Tiny heterogeneous scalar undo log. It records old bytes of trivially-copyable
// lvalues, making it convenient for hand-written delta updates across several fields.
// Do not move/reallocate the underlying objects while checkpoints are live.
class ScalarUndoLog {
  static constexpr size_t INLINE_BYTES = 32;
  struct Entry {
    void* ptr = nullptr;
    std::array<std::byte, INLINE_BYTES> old{};
    uint8_t size = 0;
  };
  std::vector<Entry> log_;
public:
  using Checkpoint = size_t;
  explicit ScalarUndoLog(size_t reserve_entries = 0) { log_.reserve(reserve_entries); }
  [[nodiscard]] Checkpoint checkpoint() const { return log_.size(); }
  [[nodiscard]] size_t size() const { return log_.size(); }
  void reserve(size_t n) { log_.reserve(n); }
  void clear() { log_.clear(); }

  template<class T>
  void save(T& ref) {
    static_assert(std::is_trivially_copyable_v<T>);
    static_assert(sizeof(T) <= INLINE_BYTES, "ScalarUndoLog supports fields up to 32 bytes");
    Entry e;
    e.ptr = static_cast<void*>(std::addressof(ref));
    e.size = static_cast<uint8_t>(sizeof(T));
    std::memcpy(e.old.data(), std::addressof(ref), sizeof(T));
    log_.push_back(e);
  }
  template<class T>
  void assign(T& ref, const T& value) {
    save(ref); ref = value;
  }
  template<class T, class U>
  void add(T& ref, const U& delta) {
    save(ref); ref += delta;
  }
  template<class T>
  void swap_values(T& a, T& b) {
    if (std::addressof(a) == std::addressof(b)) return;
    save(a); save(b); std::swap(a, b);
  }
  void rollback(Checkpoint cp) {
    assert(cp <= log_.size());
    while (log_.size() > cp) {
      const Entry& e = log_.back();
      std::memcpy(e.ptr, e.old.data(), e.size);
      log_.pop_back();
    }
  }
  // Keep mutations, discard only undo records since cp. Avoid this inside a nested
  // transaction if the outer transaction must still be able to revert those writes.
  void discard_since(Checkpoint cp) {
    assert(cp <= log_.size());
    log_.resize(cp);
  }
};

// Dense bounded 2D spatial buckets with O(1) insert/erase/move by point id.
// Best when coordinates lie in a known rectangle and local radius queries dominate.
class SpatialBuckets2D {
  double x0_ = 0, y0_ = 0, x1_ = 1, y1_ = 1, cell_ = 1;
  int nx_ = 1, ny_ = 1;
  std::vector<std::vector<int>> bucket_;
  std::vector<double> x_, y_;
  std::vector<int> bid_, pos_;
  std::vector<uint8_t> active_;

  int cx(double x) const {
    int c = static_cast<int>((x - x0_) / cell_);
    return std::clamp(c, 0, nx_ - 1);
  }
  int cy(double y) const {
    int c = static_cast<int>((y - y0_) / cell_);
    return std::clamp(c, 0, ny_ - 1);
  }
  int id_of(double x, double y) const { return cy(y) * nx_ + cx(x); }
  void erase_from_bucket(int id) {
    int b = bid_[id], p = pos_[id];
    auto& v = bucket_[b];
    int last = v.back();
    v[p] = last; pos_[last] = p;
    v.pop_back();
  }
public:
  SpatialBuckets2D() = default;
  SpatialBuckets2D(int max_points, double x0, double y0, double x1, double y1, double cell_size) {
    reset(max_points, x0, y0, x1, y1, cell_size);
  }
  void reset(int max_points, double x0, double y0, double x1, double y1, double cell_size) {
    assert(max_points >= 0 && x1 > x0 && y1 > y0 && cell_size > 0);
    x0_ = x0; y0_ = y0; x1_ = x1; y1_ = y1; cell_ = cell_size;
    nx_ = std::max(1, static_cast<int>(std::ceil((x1_ - x0_) / cell_)));
    ny_ = std::max(1, static_cast<int>(std::ceil((y1_ - y0_) / cell_)));
    bucket_.assign(static_cast<size_t>(nx_) * ny_, {});
    x_.assign(max_points, 0.0); y_.assign(max_points, 0.0);
    bid_.assign(max_points, -1); pos_.assign(max_points, -1); active_.assign(max_points, 0);
  }
  [[nodiscard]] int capacity() const { return static_cast<int>(active_.size()); }
  [[nodiscard]] bool active(int id) const { return id >= 0 && id < capacity() && active_[id]; }
  [[nodiscard]] double x(int id) const { assert(active(id)); return x_[id]; }
  [[nodiscard]] double y(int id) const { assert(active(id)); return y_[id]; }

  void insert(int id, double x, double y) {
    assert(id >= 0 && id < capacity());
    if (active_[id]) erase(id);
    int b = id_of(x, y);
    x_[id] = x; y_[id] = y; bid_[id] = b;
    pos_[id] = static_cast<int>(bucket_[b].size());
    bucket_[b].push_back(id); active_[id] = 1;
  }
  void erase(int id) {
    assert(active(id));
    erase_from_bucket(id);
    active_[id] = 0; bid_[id] = pos_[id] = -1;
  }
  void move(int id, double x, double y) {
    assert(active(id));
    int nb = id_of(x, y);
    if (nb != bid_[id]) {
      erase_from_bucket(id);
      bid_[id] = nb; pos_[id] = static_cast<int>(bucket_[nb].size());
      bucket_[nb].push_back(id);
    }
    x_[id] = x; y_[id] = y;
  }

  template<class Out>
  void query_radius(double x, double y, double r, Out& out) const {
    if (!(r >= 0.0)) return;
    int lx = std::clamp(static_cast<int>(std::floor((x - r - x0_) / cell_)), 0, nx_ - 1);
    int rx = std::clamp(static_cast<int>(std::floor((x + r - x0_) / cell_)), 0, nx_ - 1);
    int ly = std::clamp(static_cast<int>(std::floor((y - r - y0_) / cell_)), 0, ny_ - 1);
    int ry = std::clamp(static_cast<int>(std::floor((y + r - y0_) / cell_)), 0, ny_ - 1);
    double rr = r * r;
    for (int yy = ly; yy <= ry; ++yy) for (int xx = lx; xx <= rx; ++xx) {
      for (int id : bucket_[yy * nx_ + xx]) {
        double dx = x_[id] - x, dy = y_[id] - y;
        if (dx * dx + dy * dy <= rr) out.push_back(id);
      }
    }
  }
};

struct HungarianResult {
  long long cost = 0;
  std::vector<int> row_to_col;
  std::vector<int> col_to_row;
};

// Minimum-cost rectangular assignment. It matches every vertex on the smaller side.
// Costs should leave enough headroom for additions/subtractions (e.g. |cost| << 2^60).
inline HungarianResult hungarian_min_cost(const std::vector<std::vector<long long>>& cost) {
  const int n0 = static_cast<int>(cost.size());
  const int m0 = n0 ? static_cast<int>(cost[0].size()) : 0;
  for (const auto& row : cost) assert(static_cast<int>(row.size()) == m0);
  HungarianResult out;
  out.row_to_col.assign(n0, -1);
  out.col_to_row.assign(m0, -1);
  if (n0 == 0 || m0 == 0) return out;

  bool transposed = n0 > m0;
  int n = transposed ? m0 : n0;
  int m = transposed ? n0 : m0;
  auto a = [&](int i, int j) -> long long { // 0-based in normalized n x m matrix
    return transposed ? cost[j][i] : cost[i][j];
  };

  const long long INF = std::numeric_limits<long long>::max() / 4;
  std::vector<long long> u(n + 1), v(m + 1), minv(m + 1);
  std::vector<int> p(m + 1), way(m + 1);
  std::vector<uint8_t> used(m + 1);

  for (int i = 1; i <= n; ++i) {
    p[0] = i;
    int j0 = 0;
    std::fill(minv.begin(), minv.end(), INF);
    std::fill(used.begin(), used.end(), 0);
    do {
      used[j0] = 1;
      int i0 = p[j0], j1 = 0;
      long long delta = INF;
      for (int j = 1; j <= m; ++j) if (!used[j]) {
        long long cur = a(i0 - 1, j - 1) - u[i0] - v[j];
        if (cur < minv[j]) { minv[j] = cur; way[j] = j0; }
        if (minv[j] < delta) { delta = minv[j]; j1 = j; }
      }
      assert(delta < INF);
      for (int j = 0; j <= m; ++j) {
        if (used[j]) { u[p[j]] += delta; v[j] -= delta; }
        else if (j > 0) minv[j] -= delta;
      }
      j0 = j1;
    } while (p[j0] != 0);
    do {
      int j1 = way[j0];
      p[j0] = p[j1];
      j0 = j1;
    } while (j0 != 0);
  }

  out.cost = -v[0];
  for (int j = 1; j <= m; ++j) if (p[j] != 0) {
    int ri = p[j] - 1, cj = j - 1;
    if (!transposed) {
      out.row_to_col[ri] = cj;
      out.col_to_row[cj] = ri;
    } else {
      // normalized row is original column; normalized column is original row
      out.row_to_col[cj] = ri;
      out.col_to_row[ri] = cj;
    }
  }
  return out;
}

} // namespace ahc


// ===== ahc/fast_accept.hpp =====
#line 1 "ahc/fast_accept.hpp"
#include <vector>
#include <cmath>
#include <cstddef>

namespace ahc {

// Fast simulated-annealing acceptance for minimization.
// Precomputes -log(U) values so a worsening move is accepted iff
//   delta <= temperature * (-log(U)).
// This avoids exp() and a fresh floating-point RNG conversion in the hot loop.
// The table is cyclic; use a reasonably large power-of-two table (default 2^16).
class PrecomputedAnnealAccept {
  std::vector<float> neg_log_u_;
  std::size_t pos_ = 0;
public:
  explicit PrecomputedAnnealAccept(std::size_t table_size = (1u << 16),
                                   uint64_t seed = 0x6a09e667f3bcc909ULL) {
    reset(table_size, seed);
  }

  void reset(std::size_t table_size, uint64_t seed) {
    table_size = std::max<std::size_t>(1, table_size);
    neg_log_u_.resize(table_size);
    Rng rng(seed);
    for (auto& x : neg_log_u_) {
      // Keep U in (0,1], avoiding log(0).
      double u = std::max(rng.uniform01(), 0x1p-53);
      x = static_cast<float>(-std::log(u));
    }
    pos_ = static_cast<std::size_t>(rng.u64() % neg_log_u_.size());
  }

  void randomize_phase(Rng& rng) {
    pos_ = static_cast<std::size_t>(rng.u64() % neg_log_u_.size());
  }

  [[nodiscard]] std::size_t table_size() const { return neg_log_u_.size(); }

  bool accept(double delta, double temperature) {
    if (delta <= 0.0) return true;
    if (!(temperature > 0.0)) return false;
    float e = neg_log_u_[pos_];
    if (++pos_ == neg_log_u_.size()) pos_ = 0;
    return delta <= temperature * static_cast<double>(e);
  }
};

// Safe only when all remaining, not-yet-added terms are known to be >= 0.
// Useful for expensive additive candidate evaluation: stop as soon as the
// partial cost cannot beat the current threshold.
template<class Cost>
class NonnegativeCostCutoff {
  Cost value_{};
  Cost cutoff_{};
public:
  NonnegativeCostCutoff() = default;
  explicit NonnegativeCostCutoff(Cost cutoff, Cost initial = Cost{})
      : value_(initial), cutoff_(cutoff) {}

  void reset(Cost cutoff, Cost initial = Cost{}) {
    value_ = initial;
    cutoff_ = cutoff;
  }

  // Returns true while the candidate can still beat cutoff.
  bool add(Cost nonnegative_term) {
    value_ += nonnegative_term;
    return value_ < cutoff_;
  }

  [[nodiscard]] bool prunable() const { return value_ >= cutoff_; }
  [[nodiscard]] Cost value() const { return value_; }
  [[nodiscard]] Cost cutoff() const { return cutoff_; }
};

} // namespace ahc

// ===== ahc/dependency_cache.hpp =====
#line 1 "ahc/dependency_cache.hpp"
#include <vector>
#include <cstdint>
#include <cassert>
#include <utility>

namespace ahc {

// Cache derived feature values and invalidate only the features depending on a
// changed primitive. Dependency topology is static; values are computed lazily.
// Typical use: primitive = cell/vertex, feature = local penalty/edge term.
template<class T>
class DependencyCache {
  std::vector<T> value_;
  std::vector<uint32_t> stamp_;
  uint32_t epoch_ = 1;
  std::vector<std::vector<int>> reverse_dep_;

  void advance_epoch() {
    ++epoch_;
    if (epoch_ == 0) {
      std::fill(stamp_.begin(), stamp_.end(), 0);
      epoch_ = 1;
    }
  }
public:
  DependencyCache() = default;
  DependencyCache(int primitive_count, int feature_count) {
    reset_shape(primitive_count, feature_count);
  }

  void reset_shape(int primitive_count, int feature_count) {
    assert(primitive_count >= 0 && feature_count >= 0);
    value_.assign(feature_count, T{});
    stamp_.assign(feature_count, 0);
    reverse_dep_.assign(primitive_count, {});
    epoch_ = 1;
  }

  [[nodiscard]] int primitive_count() const { return static_cast<int>(reverse_dep_.size()); }
  [[nodiscard]] int feature_count() const { return static_cast<int>(value_.size()); }

  void add_dependency(int primitive, int feature) {
    assert(0 <= primitive && primitive < primitive_count());
    assert(0 <= feature && feature < feature_count());
    reverse_dep_[primitive].push_back(feature);
  }

  // Optional after building: removes duplicate dependency edges.
  void normalize_dependencies() {
    for (auto& v : reverse_dep_) {
      std::sort(v.begin(), v.end());
      v.erase(std::unique(v.begin(), v.end()), v.end());
    }
  }

  void clear() { advance_epoch(); } // O(1)

  void invalidate_feature(int feature) {
    assert(0 <= feature && feature < feature_count());
    stamp_[feature] = 0;
  }

  void invalidate_primitive(int primitive) {
    assert(0 <= primitive && primitive < primitive_count());
    for (int feature : reverse_dep_[primitive]) stamp_[feature] = 0;
  }

  template<class It>
  void invalidate_primitives(It first, It last) {
    for (; first != last; ++first) invalidate_primitive(*first);
  }

  [[nodiscard]] bool valid(int feature) const {
    assert(0 <= feature && feature < feature_count());
    return stamp_[feature] == epoch_;
  }

  template<class F>
  const T& get_or_compute(int feature, F&& compute) {
    assert(0 <= feature && feature < feature_count());
    if (stamp_[feature] != epoch_) {
      value_[feature] = std::forward<F>(compute)();
      stamp_[feature] = epoch_;
    }
    return value_[feature];
  }

  void set(int feature, const T& value) {
    assert(0 <= feature && feature < feature_count());
    value_[feature] = value;
    stamp_[feature] = epoch_;
  }
};

} // namespace ahc

// ===== ahc/min_cost_flow.hpp =====
#line 1 "ahc/min_cost_flow.hpp"
#include <vector>
#include <queue>
#include <limits>
#include <utility>
#include <cassert>

namespace ahc {

// Small/medium min-cost flow intended for LNS repair and endgames.
// Successive shortest augmenting paths with SPFA. This deliberately favors a
// short, robust API over asymptotic performance; negative edge costs are allowed
// as long as the residual graph has no reachable negative-cost cycle.
class SmallMinCostFlow {
public:
  struct Edge {
    int to = -1;
    int rev = -1;
    int cap = 0;
    long long cost = 0;
  };
  struct Result {
    int flow = 0;
    long long cost = 0;
    bool complete = false;
  };

private:
  std::vector<std::vector<Edge>> g_;

public:
  SmallMinCostFlow() = default;
  explicit SmallMinCostFlow(int n) : g_(n) {}

  void reset(int n) { g_.assign(n, {}); }
  [[nodiscard]] int size() const { return static_cast<int>(g_.size()); }

  void add_edge(int u, int v, int cap, long long cost) {
    assert(0 <= u && u < size());
    assert(0 <= v && v < size());
    assert(cap >= 0);
    Edge a{v, static_cast<int>(g_[v].size()), cap, cost};
    Edge b{u, static_cast<int>(g_[u].size()), 0, -cost};
    g_[u].push_back(a);
    g_[v].push_back(b);
  }

  const std::vector<std::vector<Edge>>& graph() const { return g_; }

  Result min_cost_flow(int s, int t, int max_flow) {
    assert(0 <= s && s < size());
    assert(0 <= t && t < size());
    assert(max_flow >= 0);
    const long long INF = std::numeric_limits<long long>::max() / 4;
    const int n = size();
    Result res;
    std::vector<long long> dist(n);
    std::vector<int> pv(n), pe(n);
    std::vector<uint8_t> inq(n);

    while (res.flow < max_flow) {
      std::fill(dist.begin(), dist.end(), INF);
      std::fill(inq.begin(), inq.end(), 0);
      std::queue<int> q;
      dist[s] = 0;
      q.push(s);
      inq[s] = 1;

      while (!q.empty()) {
        int u = q.front(); q.pop();
        inq[u] = 0;
        for (int ei = 0; ei < static_cast<int>(g_[u].size()); ++ei) {
          const Edge& e = g_[u][ei];
          if (e.cap <= 0 || dist[u] == INF) continue;
          long long nd = dist[u] + e.cost;
          if (nd < dist[e.to]) {
            dist[e.to] = nd;
            pv[e.to] = u;
            pe[e.to] = ei;
            if (!inq[e.to]) {
              inq[e.to] = 1;
              q.push(e.to);
            }
          }
        }
      }

      if (dist[t] == INF) break;
      int add = max_flow - res.flow;
      for (int v = t; v != s; v = pv[v]) {
        add = std::min(add, g_[pv[v]][pe[v]].cap);
      }
      for (int v = t; v != s; v = pv[v]) {
        Edge& e = g_[pv[v]][pe[v]];
        e.cap -= add;
        g_[v][e.rev].cap += add;
      }
      res.flow += add;
      res.cost += static_cast<long long>(add) * dist[t];
    }
    res.complete = (res.flow == max_flow);
    return res;
  }
};

} // namespace ahc


// ===== ahc/incremental_dependency_sum.hpp =====
#line 1 "ahc/incremental_dependency_sum.hpp"
#include <vector>
#include <algorithm>
#include <cassert>
#include <utility>

namespace ahc {

// Maintains an exact additive objective whose feature terms depend on a sparse
// set of mutable primitives. Mark changed primitives, then refresh only the
// affected terms. Unlike DependencyCache, this also maintains the global sum.
//
// Typical pattern:
//   IncrementalDependencySum<long long> score(cells, local_terms);
//   score.add_dependency(cell, term);
//   score.initialize([&](int term){ return recompute_term(term); });
//   ... mutate cells u,v ...
//   score.invalidate_primitive(u); score.invalidate_primitive(v);
//   long long total = score.refresh([&](int term){ return recompute_term(term); });
template<class T>
class IncrementalDependencySum {
  std::vector<T> value_;
  T total_{};
  std::vector<std::vector<int>> reverse_dep_;
  std::vector<int> dirty_list_;
  std::vector<unsigned char> dirty_;

public:
  IncrementalDependencySum() = default;
  IncrementalDependencySum(int primitive_count, int feature_count) {
    reset_shape(primitive_count, feature_count);
  }

  void reset_shape(int primitive_count, int feature_count) {
    assert(primitive_count >= 0 && feature_count >= 0);
    value_.assign(feature_count, T{});
    reverse_dep_.assign(primitive_count, {});
    dirty_.assign(feature_count, 0);
    dirty_list_.clear();
    total_ = T{};
  }

  [[nodiscard]] int primitive_count() const {
    return static_cast<int>(reverse_dep_.size());
  }
  [[nodiscard]] int feature_count() const {
    return static_cast<int>(value_.size());
  }
  [[nodiscard]] const T& total() const { return total_; }
  [[nodiscard]] const T& feature_value(int feature) const {
    assert(0 <= feature && feature < feature_count());
    return value_[feature];
  }
  [[nodiscard]] int dirty_count() const {
    return static_cast<int>(dirty_list_.size());
  }

  void add_dependency(int primitive, int feature) {
    assert(0 <= primitive && primitive < primitive_count());
    assert(0 <= feature && feature < feature_count());
    reverse_dep_[primitive].push_back(feature);
  }

  void normalize_dependencies() {
    for (auto& v : reverse_dep_) {
      std::sort(v.begin(), v.end());
      v.erase(std::unique(v.begin(), v.end()), v.end());
    }
  }

  template<class F>
  T initialize(F&& compute) {
    total_ = T{};
    std::fill(dirty_.begin(), dirty_.end(), 0);
    dirty_list_.clear();
    for (int f = 0; f < feature_count(); ++f) {
      value_[f] = compute(f);
      total_ += value_[f];
    }
    return total_;
  }

  void invalidate_feature(int feature) {
    assert(0 <= feature && feature < feature_count());
    if (!dirty_[feature]) {
      dirty_[feature] = 1;
      dirty_list_.push_back(feature);
    }
  }

  void invalidate_primitive(int primitive) {
    assert(0 <= primitive && primitive < primitive_count());
    for (int f : reverse_dep_[primitive]) invalidate_feature(f);
  }

  template<class It>
  void invalidate_primitives(It first, It last) {
    for (; first != last; ++first) invalidate_primitive(*first);
  }

  template<class F>
  T refresh(F&& compute) {
    for (int f : dirty_list_) if (dirty_[f]) {
      T nv = compute(f);
      total_ += nv - value_[f];
      value_[f] = std::move(nv);
      dirty_[f] = 0;
    }
    dirty_list_.clear();
    return total_;
  }

  // Useful when a feature delta is known directly and no recomputation is needed.
  void set_feature(int feature, T value) {
    assert(0 <= feature && feature < feature_count());
    total_ += value - value_[feature];
    value_[feature] = std::move(value);
    if (dirty_[feature]) dirty_[feature] = 0;
    // A stale id may remain in dirty_list_; refresh() safely skips it below.
  }

  template<class F>
  T refresh_skipping_clean(F&& compute) {
    for (int f : dirty_list_) if (dirty_[f]) {
      T nv = compute(f);
      total_ += nv - value_[f];
      value_[f] = std::move(nv);
      dirty_[f] = 0;
    }
    dirty_list_.clear();
    return total_;
  }
};

} // namespace ahc

// ===== ahc/implicit_sequence.hpp =====
#line 1 "ahc/implicit_sequence.hpp"
#include <vector>
#include <cassert>
#include <cstdint>
#include <utility>

namespace ahc {

// Deterministic implicit FHQ treap for sequence-heavy heuristics.
// Supports reverse / rotate / cut-and-insert in O(log N) expected time without
// touching every element. It intentionally has no generic aggregate machinery;
// route-specific objective deltas should usually stay problem-specific.
template<class T>
class ImplicitSequence {
  struct Node {
    T value{};
    uint64_t priority = 0;
    int left = -1, right = -1, size = 1;
    bool rev = false;
  };

  std::vector<Node> nodes_;
  std::vector<int> free_;
  int root_ = -1;
  uint64_t rng_state_ = 0x9e3779b97f4a7c15ULL;

  static uint64_t splitmix64(uint64_t& x) {
    uint64_t z = (x += 0x9e3779b97f4a7c15ULL);
    z = (z ^ (z >> 30)) * 0xbf58476d1ce4e5b9ULL;
    z = (z ^ (z >> 27)) * 0x94d049bb133111ebULL;
    return z ^ (z >> 31);
  }
  int sz(int x) const { return x == -1 ? 0 : nodes_[x].size; }
  void pull(int x) {
    if (x != -1) nodes_[x].size = 1 + sz(nodes_[x].left) + sz(nodes_[x].right);
  }
  void apply_rev(int x) {
    if (x == -1) return;
    std::swap(nodes_[x].left, nodes_[x].right);
    nodes_[x].rev = !nodes_[x].rev;
  }
  void push(int x) {
    if (x == -1 || !nodes_[x].rev) return;
    apply_rev(nodes_[x].left);
    apply_rev(nodes_[x].right);
    nodes_[x].rev = false;
  }
  int alloc_node(const T& value) {
    int id;
    if (free_.empty()) {
      id = static_cast<int>(nodes_.size());
      nodes_.push_back(Node{});
    } else {
      id = free_.back(); free_.pop_back();
    }
    nodes_[id] = Node{value, splitmix64(rng_state_), -1, -1, 1, false};
    return id;
  }
  int alloc_node(T&& value) {
    int id;
    if (free_.empty()) {
      id = static_cast<int>(nodes_.size());
      nodes_.push_back(Node{});
    } else {
      id = free_.back(); free_.pop_back();
    }
    nodes_[id] = Node{std::move(value), splitmix64(rng_state_), -1, -1, 1, false};
    return id;
  }

  // First k elements go to a, the rest to b.
  void split(int x, int k, int& a, int& b) {
    if (x == -1) { a = b = -1; return; }
    push(x);
    if (sz(nodes_[x].left) >= k) {
      b = x;
      split(nodes_[b].left, k, a, nodes_[b].left);
      pull(b);
    } else {
      a = x;
      int left_count = sz(nodes_[a].left) + 1;
      split(nodes_[a].right, k - left_count, nodes_[a].right, b);
      pull(a);
    }
  }

  int merge(int a, int b) {
    if (a == -1) return b;
    if (b == -1) return a;
    if (nodes_[a].priority > nodes_[b].priority) {
      push(a);
      nodes_[a].right = merge(nodes_[a].right, b);
      pull(a);
      return a;
    }
    push(b);
    nodes_[b].left = merge(a, nodes_[b].left);
    pull(b);
    return b;
  }

  T& get_impl(int x, int pos) {
    push(x);
    int ls = sz(nodes_[x].left);
    if (pos < ls) return get_impl(nodes_[x].left, pos);
    if (pos == ls) return nodes_[x].value;
    return get_impl(nodes_[x].right, pos - ls - 1);
  }
  const T& get_const_impl(int x, int pos, bool flip = false) const {
    const Node& n = nodes_[x];
    // apply_rev() already swaps this node's child pointers. n.rev means that
    // each child subtree still needs to be interpreted in reverse order.
    int l = flip ? n.right : n.left;
    int r = flip ? n.left : n.right;
    int ls = sz(l);
    bool child_flip = flip ^ n.rev;
    if (pos < ls) return get_const_impl(l, pos, child_flip);
    if (pos == ls) return n.value;
    return get_const_impl(r, pos - ls - 1, child_flip);
  }
  void collect(int x, std::vector<T>& out) {
    if (x == -1) return;
    push(x);
    collect(nodes_[x].left, out);
    out.push_back(nodes_[x].value);
    collect(nodes_[x].right, out);
  }

public:
  ImplicitSequence() = default;
  explicit ImplicitSequence(uint64_t seed) : rng_state_(seed) {}
  explicit ImplicitSequence(const std::vector<T>& values,
                            uint64_t seed = 0x9e3779b97f4a7c15ULL)
      : rng_state_(seed) { build(values); }

  void reserve(int n) { nodes_.reserve(n); free_.reserve(n); }
  void clear() { nodes_.clear(); free_.clear(); root_ = -1; }
  [[nodiscard]] int size() const { return sz(root_); }
  [[nodiscard]] bool empty() const { return root_ == -1; }

  void build(const std::vector<T>& values) {
    clear();
    reserve(static_cast<int>(values.size()));
    for (const auto& x : values) push_back(x);
  }

  void push_back(const T& value) { root_ = merge(root_, alloc_node(value)); }
  void push_back(T&& value) { root_ = merge(root_, alloc_node(std::move(value))); }

  T& at(int pos) {
    assert(0 <= pos && pos < size());
    return get_impl(root_, pos);
  }
  const T& at(int pos) const {
    assert(0 <= pos && pos < size());
    return get_const_impl(root_, pos);
  }
  void set(int pos, const T& value) { at(pos) = value; }

  void insert(int pos, const T& value) {
    assert(0 <= pos && pos <= size());
    int a, b;
    split(root_, pos, a, b);
    root_ = merge(merge(a, alloc_node(value)), b);
  }

  T erase(int pos) {
    assert(0 <= pos && pos < size());
    int a, b, c;
    split(root_, pos, a, b);
    split(b, 1, b, c);
    push(b);
    T out = std::move(nodes_[b].value);
    nodes_[b].left = nodes_[b].right = -1;
    nodes_[b].rev = false;
    nodes_[b].size = 1;
    free_.push_back(b);
    root_ = merge(a, c);
    return out;
  }

  // Reverse [l, r).
  void reverse(int l, int r) {
    assert(0 <= l && l <= r && r <= size());
    if (r - l <= 1) return;
    int a, b, c;
    split(root_, l, a, b);
    split(b, r - l, b, c);
    apply_rev(b);
    root_ = merge(merge(a, b), c);
  }

  // std::rotate semantics on [l,r): [l,m) [m,r) -> [m,r) [l,m).
  void rotate(int l, int m, int r) {
    assert(0 <= l && l <= m && m <= r && r <= size());
    if (l == m || m == r) return;
    int a, b, c, d;
    split(root_, l, a, b);
    split(b, m - l, b, c);
    split(c, r - m, c, d);
    root_ = merge(merge(merge(a, c), b), d);
  }

  // Remove [l,r), then insert the block before position pos in the remaining
  // sequence. pos is interpreted AFTER removal, so 0 <= pos <= size()-(r-l).
  void cut_insert(int l, int r, int pos) {
    assert(0 <= l && l <= r && r <= size());
    int len = r - l;
    assert(0 <= pos && pos <= size() - len);
    if (len == 0) return;
    int a, mid, c;
    split(root_, l, a, mid);
    split(mid, len, mid, c);
    int rest = merge(a, c);
    int x, y;
    split(rest, pos, x, y);
    root_ = merge(merge(x, mid), y);
  }

  std::vector<T> to_vector() {
    std::vector<T> out;
    out.reserve(size());
    collect(root_, out);
    return out;
  }
};

} // namespace ahc

// ===== ahc/route_delta.hpp =====
#line 1 "ahc/route_delta.hpp"
#include <vector>
#include <cassert>
#include <algorithm>

namespace ahc {

// O(1) move deltas for a symmetric cyclic route. The distance/cost callable
// must satisfy dist(a,b)==dist(b,a). Negative return means improvement for a
// minimization objective.
template<class Route, class Dist>
auto symmetric_two_opt_delta_cycle(const Route& route, int l, int r, Dist&& dist) {
  using Cost = decltype(dist(route[0], route[0]));
  int n = static_cast<int>(route.size());
  assert(n >= 2 && 0 <= l && l < r && r <= n);
  if (r - l <= 1 || r - l == n) return Cost{};
  int a = route[(l - 1 + n) % n];
  int b = route[l];
  int c = route[r - 1];
  int d = route[r % n];
  return dist(a, c) + dist(b, d) - dist(a, b) - dist(c, d);
}

template<class Route, class Dist>
auto symmetric_swap_delta_cycle(const Route& route, int i, int j, Dist&& dist) {
  using Cost = decltype(dist(route[0], route[0]));
  int n = static_cast<int>(route.size());
  assert(n >= 2 && 0 <= i && i < n && 0 <= j && j < n);
  if (i == j) return Cost{};
  int starts[4] = {(i - 1 + n) % n, i, (j - 1 + n) % n, j};
  std::sort(starts, starts + 4);
  int m = static_cast<int>(std::unique(starts, starts + 4) - starts);
  auto value_after = [&](int p) {
    if (p == i) return route[j];
    if (p == j) return route[i];
    return route[p];
  };
  Cost old_cost{}, new_cost{};
  for (int k = 0; k < m; ++k) {
    int p = starts[k], q = (p + 1) % n;
    old_cost += dist(route[p], route[q]);
    new_cost += dist(value_after(p), value_after(q));
  }
  return new_cost - old_cost;
}

// Move route[from] to immediately after route[after], where indices refer to
// the ORIGINAL cycle. No-op if that node is already immediately after `after`.
template<class Route, class Dist>
auto symmetric_relocate_one_delta_cycle(const Route& route, int from, int after,
                                         Dist&& dist) {
  using Cost = decltype(dist(route[0], route[0]));
  int n = static_cast<int>(route.size());
  assert(n >= 3 && 0 <= from && from < n && 0 <= after && after < n);
  if (from == after || (after + 1) % n == from) return Cost{};
  int a = route[(from - 1 + n) % n];
  int x = route[from];
  int b = route[(from + 1) % n];
  int y = route[after];
  int z = route[(after + 1) % n];
  return dist(a, b) + dist(y, x) + dist(x, z)
       - dist(a, x) - dist(x, b) - dist(y, z);
}

} // namespace ahc

// ===== ahc/potential_min_cost_flow.hpp =====
#line 1 "ahc/potential_min_cost_flow.hpp"
#include <vector>
#include <queue>
#include <limits>
#include <cassert>
#include <utility>
#include <functional>

namespace ahc {

// Successive-shortest-path min-cost flow with Johnson potentials.
// Compared with SmallMinCostFlow (SPFA per augmentation), this pays one initial
// shortest-path pass when negative edges exist, then uses Dijkstra. It is a good
// fit for repair networks that need many augmentations.
class PotentialMinCostFlow {
public:
  struct Edge {
    int to = -1;
    int rev = -1;
    int cap = 0;
    long long cost = 0;
  };
  struct Result {
    int flow = 0;
    long long cost = 0;
    bool complete = false;
  };

private:
  std::vector<std::vector<Edge>> g_;
  bool has_negative_cost_ = false;

public:
  PotentialMinCostFlow() = default;
  explicit PotentialMinCostFlow(int n) : g_(n) {}
  void reset(int n) { g_.assign(n, {}); has_negative_cost_ = false; }
  [[nodiscard]] int size() const { return static_cast<int>(g_.size()); }

  void add_edge(int u, int v, int cap, long long cost) {
    assert(0 <= u && u < size() && 0 <= v && v < size() && cap >= 0);
    Edge a{v, static_cast<int>(g_[v].size()), cap, cost};
    Edge b{u, static_cast<int>(g_[u].size()), 0, -cost};
    g_[u].push_back(a);
    g_[v].push_back(b);
    if (cap > 0 && cost < 0) has_negative_cost_ = true;
  }

  const std::vector<std::vector<Edge>>& graph() const { return g_; }

  Result min_cost_flow(int s, int t, int max_flow) {
    assert(0 <= s && s < size() && 0 <= t && t < size() && max_flow >= 0);
    const int n = size();
    const long long INF = std::numeric_limits<long long>::max() / 4;
    std::vector<long long> pot(n, 0), dist(n);
    std::vector<int> pv(n), pe(n);

    if (has_negative_cost_) {
      // Initial shortest distances. SPFA is paid once; afterwards reduced costs
      // are nonnegative as long as there is no reachable negative-cost cycle.
      std::vector<unsigned char> inq(n, 0);
      std::queue<int> q;
      std::fill(dist.begin(), dist.end(), INF);
      dist[s] = 0; q.push(s); inq[s] = 1;
      while (!q.empty()) {
        int u = q.front(); q.pop(); inq[u] = 0;
        for (const auto& e : g_[u]) if (e.cap > 0 && dist[u] != INF) {
          long long nd = dist[u] + e.cost;
          if (nd < dist[e.to]) {
            dist[e.to] = nd;
            if (!inq[e.to]) { inq[e.to] = 1; q.push(e.to); }
          }
        }
      }
      for (int v = 0; v < n; ++v) if (dist[v] != INF) pot[v] = dist[v];
    }

    Result res;
    using QN = std::pair<long long, int>;
    while (res.flow < max_flow) {
      std::fill(dist.begin(), dist.end(), INF);
      std::priority_queue<QN, std::vector<QN>, std::greater<QN>> pq;
      dist[s] = 0; pq.push({0, s});
      while (!pq.empty()) {
        auto [du, u] = pq.top(); pq.pop();
        if (du != dist[u]) continue;
        for (int ei = 0; ei < static_cast<int>(g_[u].size()); ++ei) {
          const Edge& e = g_[u][ei];
          if (e.cap <= 0) continue;
          long long rcost = e.cost + pot[u] - pot[e.to];
          // Floating issues do not exist; a negative reduced cost here signals
          // an invalid potential (typically a reachable negative cycle).
          long long nd = du + rcost;
          if (nd < dist[e.to]) {
            dist[e.to] = nd;
            pv[e.to] = u;
            pe[e.to] = ei;
            pq.push({nd, e.to});
          }
        }
      }
      if (dist[t] == INF) break;
      for (int v = 0; v < n; ++v) if (dist[v] != INF) pot[v] += dist[v];

      int add = max_flow - res.flow;
      for (int v = t; v != s; v = pv[v])
        add = std::min(add, g_[pv[v]][pe[v]].cap);
      for (int v = t; v != s; v = pv[v]) {
        Edge& e = g_[pv[v]][pe[v]];
        e.cap -= add;
        g_[v][e.rev].cap += add;
      }
      res.flow += add;
      res.cost += static_cast<long long>(add) * (pot[t] - pot[s]);
    }
    res.complete = (res.flow == max_flow);
    return res;
  }
};

} // namespace ahc

// ===== ahc/aggregated_implicit_sequence.hpp =====
#line 1 "ahc/aggregated_implicit_sequence.hpp"
#include <vector>
#include <cassert>
#include <cstdint>
#include <utility>
#include <type_traits>

namespace ahc {

// Implicit FHQ treap with a generic associative range aggregate.
// Stores both forward and reverse aggregates, so lazy reversal remains O(1).
// Merge must be associative; identity must be a neutral element.
// Prefer the plain ImplicitSequence when no aggregate query is needed.
template<class T, class Agg, class Lift, class Merge>
class AggregatedImplicitSequence {
  struct Node {
    T value{};
    Agg fwd{};
    Agg rev_agg{};
    uint64_t priority = 0;
    int left = -1, right = -1, size = 1;
    bool rev = false;
  };

  std::vector<Node> nodes_;
  std::vector<int> free_;
  int root_ = -1;
  uint64_t rng_state_ = 0x9e3779b97f4a7c15ULL;
  Agg identity_{};
  Lift lift_;
  Merge merge_;

  static uint64_t splitmix64(uint64_t& x) {
    uint64_t z = (x += 0x9e3779b97f4a7c15ULL);
    z = (z ^ (z >> 30)) * 0xbf58476d1ce4e5b9ULL;
    z = (z ^ (z >> 27)) * 0x94d049bb133111ebULL;
    return z ^ (z >> 31);
  }
  int sz(int x) const { return x == -1 ? 0 : nodes_[x].size; }
  const Agg& fwd_of(int x) const { return x == -1 ? identity_ : nodes_[x].fwd; }
  const Agg& rev_of(int x) const { return x == -1 ? identity_ : nodes_[x].rev_agg; }

  void pull(int x) {
    if (x == -1) return;
    Node& n = nodes_[x];
    n.size = 1 + sz(n.left) + sz(n.right);
    Agg mid = lift_(n.value);
    n.fwd = merge_(merge_(fwd_of(n.left), mid), fwd_of(n.right));
    n.rev_agg = merge_(merge_(rev_of(n.right), std::move(mid)), rev_of(n.left));
  }
  void apply_rev(int x) {
    if (x == -1) return;
    Node& n = nodes_[x];
    std::swap(n.left, n.right);
    std::swap(n.fwd, n.rev_agg);
    n.rev = !n.rev;
  }
  void push(int x) {
    if (x == -1 || !nodes_[x].rev) return;
    apply_rev(nodes_[x].left);
    apply_rev(nodes_[x].right);
    nodes_[x].rev = false;
  }
  template<class U>
  int alloc_node(U&& value) {
    int id;
    if (free_.empty()) {
      id = static_cast<int>(nodes_.size());
      nodes_.push_back(Node{});
    } else {
      id = free_.back(); free_.pop_back();
    }
    Node n;
    n.value = std::forward<U>(value);
    n.fwd = n.rev_agg = lift_(n.value);
    n.priority = splitmix64(rng_state_);
    nodes_[id] = std::move(n);
    return id;
  }
  void split(int x, int k, int& a, int& b) {
    if (x == -1) { a = b = -1; return; }
    push(x);
    if (sz(nodes_[x].left) >= k) {
      b = x;
      split(nodes_[b].left, k, a, nodes_[b].left);
      pull(b);
    } else {
      a = x;
      int left_count = sz(nodes_[a].left) + 1;
      split(nodes_[a].right, k - left_count, nodes_[a].right, b);
      pull(a);
    }
  }
  int merge_nodes(int a, int b) {
    if (a == -1) return b;
    if (b == -1) return a;
    if (nodes_[a].priority > nodes_[b].priority) {
      push(a);
      nodes_[a].right = merge_nodes(nodes_[a].right, b);
      pull(a);
      return a;
    }
    push(b);
    nodes_[b].left = merge_nodes(a, nodes_[b].left);
    pull(b);
    return b;
  }
  void collect(int x, std::vector<T>& out) {
    if (x == -1) return;
    push(x);
    collect(nodes_[x].left, out);
    out.push_back(nodes_[x].value);
    collect(nodes_[x].right, out);
  }

public:
  AggregatedImplicitSequence(Agg identity, Lift lift, Merge merge,
                             uint64_t seed = 0x9e3779b97f4a7c15ULL)
      : rng_state_(seed), identity_(std::move(identity)),
        lift_(std::move(lift)), merge_(std::move(merge)) {}

  AggregatedImplicitSequence(const std::vector<T>& values, Agg identity,
                             Lift lift, Merge merge,
                             uint64_t seed = 0x9e3779b97f4a7c15ULL)
      : AggregatedImplicitSequence(std::move(identity), std::move(lift),
                                   std::move(merge), seed) {
    build(values);
  }

  void reserve(int n) { nodes_.reserve(n); free_.reserve(n); }
  void clear() { nodes_.clear(); free_.clear(); root_ = -1; }
  [[nodiscard]] int size() const { return sz(root_); }
  [[nodiscard]] bool empty() const { return root_ == -1; }
  [[nodiscard]] const Agg& aggregate_all() const {
    return root_ == -1 ? identity_ : nodes_[root_].fwd;
  }

  void build(const std::vector<T>& values) {
    clear(); reserve(static_cast<int>(values.size()));
    for (const auto& v : values) push_back(v);
  }
  void push_back(const T& value) { root_ = merge_nodes(root_, alloc_node(value)); }
  void push_back(T&& value) { root_ = merge_nodes(root_, alloc_node(std::move(value))); }

  void insert(int pos, const T& value) {
    assert(0 <= pos && pos <= size());
    int a, b; split(root_, pos, a, b);
    root_ = merge_nodes(merge_nodes(a, alloc_node(value)), b);
  }
  T erase(int pos) {
    assert(0 <= pos && pos < size());
    int a, b, c; split(root_, pos, a, b); split(b, 1, b, c);
    push(b); T out = std::move(nodes_[b].value);
    nodes_[b].left = nodes_[b].right = -1; nodes_[b].size = 1;
    nodes_[b].rev = false; nodes_[b].fwd = nodes_[b].rev_agg = identity_;
    free_.push_back(b); root_ = merge_nodes(a, c); return out;
  }
  void set(int pos, const T& value) {
    assert(0 <= pos && pos < size());
    int a, b, c; split(root_, pos, a, b); split(b, 1, b, c);
    push(b); nodes_[b].value = value; pull(b);
    root_ = merge_nodes(merge_nodes(a, b), c);
  }

  void reverse(int l, int r) {
    assert(0 <= l && l <= r && r <= size());
    if (r - l <= 1) return;
    int a, b, c; split(root_, l, a, b); split(b, r-l, b, c);
    apply_rev(b); root_ = merge_nodes(merge_nodes(a, b), c);
  }
  void rotate(int l, int m, int r) {
    assert(0 <= l && l <= m && m <= r && r <= size());
    if (l == m || m == r) return;
    int a, b, c, d;
    split(root_, l, a, b); split(b, m-l, b, c); split(c, r-m, c, d);
    root_ = merge_nodes(merge_nodes(merge_nodes(a, c), b), d);
  }
  void cut_insert(int l, int r, int pos) {
    assert(0 <= l && l <= r && r <= size());
    int len = r-l; assert(0 <= pos && pos <= size()-len);
    if (!len) return;
    int a, mid, c; split(root_, l, a, mid); split(mid, len, mid, c);
    int rest = merge_nodes(a, c), x, y; split(rest, pos, x, y);
    root_ = merge_nodes(merge_nodes(x, mid), y);
  }

  Agg aggregate(int l, int r) {
    assert(0 <= l && l <= r && r <= size());
    if (l == r) return identity_;
    int a, b, c; split(root_, l, a, b); split(b, r-l, b, c);
    Agg ans = b == -1 ? identity_ : nodes_[b].fwd;
    root_ = merge_nodes(merge_nodes(a, b), c);
    return ans;
  }

  std::vector<T> to_vector() {
    std::vector<T> out; out.reserve(size()); collect(root_, out); return out;
  }
};

template<class T, class Agg, class Lift, class Merge>
auto make_aggregated_implicit_sequence(const std::vector<T>& values, Agg identity,
                                       Lift lift, Merge merge,
                                       uint64_t seed = 0x9e3779b97f4a7c15ULL) {
  return AggregatedImplicitSequence<T, Agg, Lift, Merge>(
      values, std::move(identity), std::move(lift), std::move(merge), seed);
}

} // namespace ahc

// ===== ahc/sparse_dirty_propagation.hpp =====
#line 1 "ahc/sparse_dirty_propagation.hpp"
#include <vector>
#include <cassert>
#include <algorithm>

namespace ahc {

// Sparse worklist for cascading local recomputation.
// Typical use: mark changed derived nodes; pop one, recompute it, and if its
// value changed call mark_dependents(v). Each node is queued at most once until popped.
class SparseDirtyPropagation {
  std::vector<std::vector<int>> out_;
  std::vector<unsigned char> dirty_;
  std::vector<int> q_;
  std::size_t head_ = 0;
public:
  SparseDirtyPropagation() = default;
  explicit SparseDirtyPropagation(int n) { reset(n); }
  void reset(int n) {
    assert(n >= 0); out_.assign(n, {}); dirty_.assign(n, 0); q_.clear(); head_ = 0;
  }
  [[nodiscard]] int size() const { return static_cast<int>(out_.size()); }
  void add_edge(int u, int v) {
    assert(0 <= u && u < size() && 0 <= v && v < size());
    out_[u].push_back(v);
  }
  void normalize() {
    for (auto& a : out_) { std::sort(a.begin(), a.end()); a.erase(std::unique(a.begin(), a.end()), a.end()); }
  }
  void mark(int v) {
    assert(0 <= v && v < size());
    if (!dirty_[v]) { dirty_[v] = 1; q_.push_back(v); }
  }
  template<class It> void mark_many(It first, It last) { for (; first != last; ++first) mark(*first); }
  void mark_dependents(int v) {
    assert(0 <= v && v < size());
    for (int to : out_[v]) mark(to);
  }
  [[nodiscard]] bool empty() const { return head_ >= q_.size(); }
  [[nodiscard]] int pending() const { return static_cast<int>(q_.size() - head_); }
  [[nodiscard]] bool is_dirty(int v) const { assert(0 <= v && v < size()); return dirty_[v] != 0; }
  int pop() {
    assert(!empty()); int v = q_[head_++]; dirty_[v] = 0;
    if (head_ == q_.size()) { q_.clear(); head_ = 0; }
    else if (head_ > 4096 && head_ * 2 > q_.size()) {
      q_.erase(q_.begin(), q_.begin() + static_cast<std::ptrdiff_t>(head_)); head_ = 0;
    }
    return v;
  }
  void clear() {
    for (std::size_t i = head_; i < q_.size(); ++i) dirty_[q_[i]] = 0;
    q_.clear(); head_ = 0;
  }
};

} // namespace ahc

// ===== ahc/static_kdtree2d.hpp =====
#line 1 "ahc/static_kdtree2d.hpp"
#include <vector>
#include <queue>
#include <algorithm>
#include <cmath>
#include <limits>
#include <cassert>
#include <utility>

namespace ahc {

struct KDPoint2D {
  double x = 0, y = 0;
  int id = -1;
};

// Static 2D kd-tree for nearest-neighbor candidate generation.
// Use SpatialBuckets2D when points move frequently; use this when points are
// mostly fixed and k-nearest/radius queries dominate.
class StaticKDTree2D {
  struct Node {
    KDPoint2D p;
    int left = -1, right = -1;
    double xmin = 0, xmax = 0, ymin = 0, ymax = 0;
  };
  std::vector<Node> nodes_;
  int root_ = -1;

  static double sqr(double x) { return x*x; }
  static double point_d2(const KDPoint2D& a, double x, double y) {
    return sqr(a.x-x) + sqr(a.y-y);
  }
  double box_d2(int v, double x, double y) const {
    if (v == -1) return std::numeric_limits<double>::infinity();
    const Node& n = nodes_[v];
    double dx = x < n.xmin ? n.xmin-x : (x > n.xmax ? x-n.xmax : 0.0);
    double dy = y < n.ymin ? n.ymin-y : (y > n.ymax ? y-n.ymax : 0.0);
    return dx*dx + dy*dy;
  }
  int build_rec(std::vector<KDPoint2D>& pts, int l, int r, int depth) {
    if (l >= r) return -1;
    int m = (l+r)/2, axis = depth & 1;
    std::nth_element(pts.begin()+l, pts.begin()+m, pts.begin()+r,
      [axis](const auto& a, const auto& b){ return axis ? a.y < b.y : a.x < b.x; });
    int id = static_cast<int>(nodes_.size()); nodes_.push_back(Node{});
    nodes_[id].p = pts[m];
    int L = build_rec(pts, l, m, depth+1), R = build_rec(pts, m+1, r, depth+1);
    nodes_[id].left = L; nodes_[id].right = R;
    Node& n = nodes_[id];
    n.xmin = n.xmax = n.p.x; n.ymin = n.ymax = n.p.y;
    auto absorb = [&](int c){ if (c==-1) return; const Node& z=nodes_[c];
      n.xmin=std::min(n.xmin,z.xmin); n.xmax=std::max(n.xmax,z.xmax);
      n.ymin=std::min(n.ymin,z.ymin); n.ymax=std::max(n.ymax,z.ymax); };
    absorb(L); absorb(R); return id;
  }
  using HeapItem = std::pair<double,int>; // d2,id, max-heap
  void knn_rec(int v, double x, double y, int k, int exclude,
               std::priority_queue<HeapItem>& heap) const {
    if (v == -1) return;
    const Node& n = nodes_[v];
    if (n.p.id != exclude) {
      double d2 = point_d2(n.p,x,y);
      if ((int)heap.size() < k) heap.push({d2,n.p.id});
      else if (d2 < heap.top().first || (d2 == heap.top().first && n.p.id < heap.top().second)) {
        heap.pop(); heap.push({d2,n.p.id});
      }
    }
    int a=n.left,b=n.right;
    if (box_d2(a,x,y) > box_d2(b,x,y)) std::swap(a,b);
    if (a!=-1 && ((int)heap.size()<k || box_d2(a,x,y) <= heap.top().first)) knn_rec(a,x,y,k,exclude,heap);
    if (b!=-1 && ((int)heap.size()<k || box_d2(b,x,y) <= heap.top().first)) knn_rec(b,x,y,k,exclude,heap);
  }
  void radius_rec(int v, double x, double y, double r2, int exclude,
                  std::vector<int>& out) const {
    if (v==-1 || box_d2(v,x,y) > r2) return;
    const Node& n=nodes_[v];
    if (n.p.id != exclude && point_d2(n.p,x,y) <= r2) out.push_back(n.p.id);
    radius_rec(n.left,x,y,r2,exclude,out); radius_rec(n.right,x,y,r2,exclude,out);
  }

public:
  StaticKDTree2D() = default;
  explicit StaticKDTree2D(std::vector<KDPoint2D> pts) { build(std::move(pts)); }
  void build(std::vector<KDPoint2D> pts) {
    nodes_.clear(); nodes_.reserve(pts.size()); root_ = build_rec(pts,0,(int)pts.size(),0);
  }
  [[nodiscard]] int size() const { return static_cast<int>(nodes_.size()); }

  std::vector<int> k_nearest(double x, double y, int k, int exclude_id=-1) const {
    if (k <= 0 || root_ == -1) return {};
    k = std::min(k, size() - (exclude_id==-1 ? 0 : 1));
    if (k <= 0) return {};
    std::priority_queue<HeapItem> heap; knn_rec(root_,x,y,k,exclude_id,heap);
    std::vector<std::pair<double,int>> tmp; tmp.reserve(heap.size());
    while (!heap.empty()) { tmp.push_back(heap.top()); heap.pop(); }
    std::sort(tmp.begin(), tmp.end(), [](auto a, auto b){ return a.first!=b.first ? a.first<b.first : a.second<b.second; });
    std::vector<int> out; out.reserve(tmp.size()); for (auto [d,id]:tmp) out.push_back(id); return out;
  }
  std::vector<int> radius(double x, double y, double r, int exclude_id=-1) const {
    assert(r >= 0); std::vector<int> out; radius_rec(root_,x,y,r*r,exclude_id,out); return out;
  }
  std::vector<std::vector<int>> build_knn_graph(const std::vector<KDPoint2D>& original,
                                                int k, bool make_symmetric=true) const {
    std::vector<std::vector<int>> g(original.size());
    for (const auto& p : original) {
      assert(0 <= p.id && p.id < (int)g.size());
      g[p.id] = k_nearest(p.x,p.y,k,p.id);
    }
    if (make_symmetric) {
      for (int u=0;u<(int)g.size();++u) for (int v:g[u]) g[v].push_back(u);
      for (auto& a:g) { std::sort(a.begin(),a.end()); a.erase(std::unique(a.begin(),a.end()),a.end()); }
    }
    return g;
  }
};

} // namespace ahc

// ===== ahc/route_delta_extra.hpp =====
#line 1 "ahc/route_delta_extra.hpp"
#include <cassert>
#include <array>
#include <algorithm>

namespace ahc {

// Relocate a non-wrapping segment [l,r) in a symmetric cycle. `pos` is the
// insertion position in the sequence AFTER removal (insert before remaining[pos]).
// O(1) delta; requires at least two vertices outside the moved block.
template<class Route, class Dist>
auto symmetric_relocate_segment_delta_cycle(const Route& route, int l, int r,
                                              int pos, Dist&& dist) {
  int n = static_cast<int>(route.size());
  assert(n >= 4 && 0 <= l && l < r && r <= n);
  int len = r-l, m=n-len; assert(m >= 2 && 0 <= pos && pos < m);
  auto orig_of_remaining = [&](int j) { j%=m; if (j<0) j+=m; return j < l ? j : j+len; };
  int A=route[(l-1+n)%n], B=route[l], C=route[r-1], D=route[r%n];
  int V=route[orig_of_remaining(pos)], U=route[orig_of_remaining(pos-1)];
  return dist(A,D)+dist(U,B)+dist(C,V)-dist(A,B)-dist(C,D)-dist(U,V);
}

struct ThreeOptChoice {
  bool swap_segments = false;
  bool reverse_first = false;
  bool reverse_second = false;
};

// Seven nontrivial symmetric 3-opt reconnections after cutting edges
// (i,i+1), (j,j+1), (k,k+1), with 0<=i<j<k<n-1.
// The two middle segments are S1=[i+1..j], S2=[j+1..k]. S3 is fixed.
// choice describes whether S1/S2 are swapped and independently reversed.
template<class Route, class Dist>
auto symmetric_three_opt_deltas_cycle(const Route& route, int i, int j, int k,
                                       Dist&& dist) {
  using Cost = decltype(dist(route[0], route[0]));
  int n=(int)route.size(); assert(n>=6 && 0<=i && i<j && j<k && k<n-1);
  int A=route[i], B=route[i+1], C=route[j], D=route[j+1], E=route[k], F=route[k+1];
  Cost old = dist(A,B)+dist(C,D)+dist(E,F);
  std::array<std::pair<ThreeOptChoice,Cost>,7> out{}; int z=0;
  for (int sw=0;sw<2;++sw) for (int r1=0;r1<2;++r1) for (int r2=0;r2<2;++r2) {
    if (!sw && !r1 && !r2) continue; // original
    // S1 endpoints under orientation, S2 endpoints under orientation.
    int s1a = r1 ? C : B, s1b = r1 ? B : C;
    int s2a = r2 ? E : D, s2b = r2 ? D : E;
    int x1a,x1b,x2a,x2b;
    if (!sw) { x1a=s1a; x1b=s1b; x2a=s2a; x2b=s2b; }
    else { x1a=s2a; x1b=s2b; x2a=s1a; x2b=s1b; }
    Cost nw = dist(A,x1a)+dist(x1b,x2a)+dist(x2b,F);
    out[z++]={{bool(sw),bool(r1),bool(r2)}, nw-old};
  }
  return out;
}

} // namespace ahc

// ===== ahc/reusable_potential_min_cost_flow.hpp =====
#line 1 "ahc/reusable_potential_min_cost_flow.hpp"
#include <vector>
#include <queue>
#include <limits>
#include <cassert>
#include <functional>
#include <utility>

namespace ahc {

// Reusable topology min-cost flow. Edge handles let repair code change
// capacities/costs without rebuilding adjacency vectors. solve(..., true)
// restores base capacities first. Work arrays are retained across calls.
class ReusablePotentialMinCostFlow {
public:
  struct Edge { int to=-1, rev=-1, cap=0, base_cap=0; long long cost=0; };
  struct EdgeHandle { int from=-1, index=-1; };
  struct Result { int flow=0; long long cost=0; bool complete=false; };
private:
  std::vector<std::vector<Edge>> g_;
  std::vector<long long> pot_, dist_;
  std::vector<int> pv_, pe_;
  std::vector<unsigned char> inq_;
public:
  ReusablePotentialMinCostFlow() = default;
  explicit ReusablePotentialMinCostFlow(int n) { reset(n); }
  void reset(int n) {
    g_.assign(n,{}); pot_.assign(n,0); dist_.resize(n); pv_.resize(n); pe_.resize(n); inq_.resize(n);
  }
  [[nodiscard]] int size() const { return (int)g_.size(); }
  EdgeHandle add_edge(int u,int v,int cap,long long cost) {
    assert(0<=u&&u<size()&&0<=v&&v<size()&&cap>=0);
    EdgeHandle h{u,(int)g_[u].size()};
    Edge a{v,(int)g_[v].size(),cap,cap,cost}; Edge b{u,(int)g_[u].size(),0,0,-cost};
    g_[u].push_back(a); g_[v].push_back(b); return h;
  }
  void set_capacity(EdgeHandle h,int cap) {
    assert(0<=h.from&&h.from<size()&&0<=h.index&&h.index<(int)g_[h.from].size()&&cap>=0);
    Edge& e=g_[h.from][h.index]; e.base_cap=cap; e.cap=cap; g_[e.to][e.rev].cap=g_[e.to][e.rev].base_cap=0;
  }
  void set_cost(EdgeHandle h,long long cost) {
    Edge& e=g_[h.from][h.index]; e.cost=cost; g_[e.to][e.rev].cost=-cost;
  }
  void reset_residual() {
    for (auto& a:g_) for (auto& e:a) e.cap=e.base_cap;
  }
  const std::vector<std::vector<Edge>>& graph() const { return g_; }

  Result min_cost_flow(int s,int t,int max_flow,bool reset_before=true) {
    assert(0<=s&&s<size()&&0<=t&&t<size()&&max_flow>=0);
    if (reset_before) reset_residual();
    const int n=size(); const long long INF=std::numeric_limits<long long>::max()/4;
    std::fill(pot_.begin(),pot_.end(),0);
    bool neg=false; for (int u=0;u<n&&!neg;++u) for (const auto& e:g_[u]) if(e.cap>0&&e.cost<0){neg=true;break;}
    if (neg) {
      std::fill(dist_.begin(),dist_.end(),INF); std::fill(inq_.begin(),inq_.end(),0);
      std::queue<int> q; dist_[s]=0; q.push(s); inq_[s]=1;
      while(!q.empty()) { int u=q.front();q.pop();inq_[u]=0;
        for(const auto& e:g_[u]) if(e.cap>0&&dist_[u]!=INF) { long long nd=dist_[u]+e.cost; if(nd<dist_[e.to]){dist_[e.to]=nd;if(!inq_[e.to]){inq_[e.to]=1;q.push(e.to);}} }
      }
      for(int v=0;v<n;++v) if(dist_[v]!=INF) pot_[v]=dist_[v];
    }
    Result res; using Q=std::pair<long long,int>;
    while(res.flow<max_flow) {
      std::fill(dist_.begin(),dist_.end(),INF); std::priority_queue<Q,std::vector<Q>,std::greater<Q>> pq;
      dist_[s]=0; pq.push({0,s});
      while(!pq.empty()) { auto [du,u]=pq.top();pq.pop(); if(du!=dist_[u])continue;
        for(int ei=0;ei<(int)g_[u].size();++ei){const Edge&e=g_[u][ei];if(e.cap<=0)continue; long long nd=du+e.cost+pot_[u]-pot_[e.to]; if(nd<dist_[e.to]){dist_[e.to]=nd;pv_[e.to]=u;pe_[e.to]=ei;pq.push({nd,e.to});}}
      }
      if (dist_[t] == INF) break;
      for (int v=0; v<n; ++v) if (dist_[v] != INF) pot_[v] += dist_[v];
      int add=max_flow-res.flow; for(int v=t;v!=s;v=pv_[v])add=std::min(add,g_[pv_[v]][pe_[v]].cap);
      for(int v=t;v!=s;v=pv_[v]){Edge&e=g_[pv_[v]][pe_[v]];e.cap-=add;g_[v][e.rev].cap+=add;}
      res.flow+=add; res.cost+=(long long)add*(pot_[t]-pot_[s]);
    }
    res.complete=res.flow==max_flow; return res;
  }
};

} // namespace ahc


// ===== ahc/touched_set.hpp =====
#line 1 "ahc/touched_set.hpp"
#include <vector>
#include <cassert>

namespace ahc {

// Sparse unique set for indices that are "touched" during one local update.
// add/contains are O(1), clear is O(1) amortized via StampedArray generation.
class TouchedSet {
  StampedArray<unsigned char> seen_;
  std::vector<int> items_;
public:
  TouchedSet() = default;
  explicit TouchedSet(int n) : seen_(n) { items_.reserve(n); }
  void reset(int n) { seen_.reset(n); items_.clear(); items_.reserve(n); }
  void clear() { seen_.clear(); items_.clear(); }
  [[nodiscard]] int universe_size() const { return seen_.size(); }
  [[nodiscard]] int size() const { return static_cast<int>(items_.size()); }
  [[nodiscard]] bool empty() const { return items_.empty(); }
  [[nodiscard]] bool contains(int x) const { return seen_.contains(x); }
  bool add(int x) {
    assert(0 <= x && x < seen_.size());
    if (seen_.contains(x)) return false;
    seen_.set(x, 1);
    items_.push_back(x);
    return true;
  }
  const std::vector<int>& items() const { return items_; }
  auto begin() const { return items_.begin(); }
  auto end() const { return items_.end(); }
};

} // namespace ahc

// ===== ahc/distinct_sampler.hpp =====
#line 1 "ahc/distinct_sampler.hpp"
#include <vector>
#include <cassert>

namespace ahc {

// Draws a random permutation prefix of [0,n) without O(n) initialization.
// begin(n) is O(1) if n is unchanged, each draw is expected/O(1), and drawing
// k distinct indices costs O(k) memory writes rather than shuffling all n.
class DistinctSampler {
  int n_ = 0;
  int drawn_ = 0;
  StampedArray<int> remap_;

  int mapped(int x) const {
    return remap_.contains(x) ? remap_.get(x) : x;
  }
public:
  DistinctSampler() = default;
  explicit DistinctSampler(int n) { reset(n); }
  void reset(int n) {
    assert(n >= 0);
    n_ = n; drawn_ = 0; remap_.reset(n);
  }
  void begin() { drawn_ = 0; remap_.clear(); }
  void begin(int n) {
    assert(n >= 0);
    if (n != n_) reset(n);
    else begin();
  }
  [[nodiscard]] int size() const { return n_; }
  [[nodiscard]] int drawn() const { return drawn_; }
  [[nodiscard]] int remaining() const { return n_ - drawn_; }
  int draw(Rng& rng) {
    assert(drawn_ < n_);
    int m = n_ - drawn_;
    int j = rng.int_range(0, m);
    int v = mapped(j);
    int last = mapped(m - 1);
    if (j != m - 1) remap_.set(j, last);
    ++drawn_;
    return v;
  }
  std::vector<int> sample(int k, Rng& rng) {
    assert(0 <= k && k <= n_);
    begin();
    std::vector<int> out; out.reserve(k);
    for (int i = 0; i < k; ++i) out.push_back(draw(rng));
    return out;
  }
};

} // namespace ahc

// ===== ahc/dijkstra_workspace.hpp =====
#line 1 "ahc/dijkstra_workspace.hpp"
#include <queue>
#include <vector>
#include <limits>
#include <functional>
#include <span>
#include <utility>
#include <type_traits>
#include <cassert>

namespace ahc {

// Reusable Dijkstra workspace with O(1)-clear stamped distances/parents.
// Expand signature: expand(u, relax), where relax(v, nonnegative_cost).
template<class Cost>
class DijkstraWorkspace {
  StampedArray<Cost> dist_;
  StampedArray<int> parent_;
  int n_ = 0;

  static constexpr Cost inf_value() {
    return std::numeric_limits<Cost>::max() / Cost(4);
  }
public:
  DijkstraWorkspace() = default;
  explicit DijkstraWorkspace(int n) { reset(n); }
  void reset(int n) {
    assert(n >= 0);
    n_ = n; dist_.reset(n); parent_.reset(n);
  }
  [[nodiscard]] int size() const { return n_; }
  [[nodiscard]] bool reached(int v) const { return dist_.contains(v); }
  [[nodiscard]] Cost distance(int v) const { return dist_.contains(v) ? dist_.get(v) : inf_value(); }
  [[nodiscard]] int parent(int v) const { return parent_.contains(v) ? parent_.get(v) : -1; }

  template<class Expand>
  Cost run(int n, int source, int target, Expand&& expand,
           Cost max_distance = inf_value()) {
    std::pair<int,Cost> src{source, Cost(0)};
    return run_multi(n, std::span<const std::pair<int,Cost>>(&src, 1), target,
                     std::forward<Expand>(expand), max_distance);
  }

  template<class Expand>
  Cost run_all(int n, int source, Expand&& expand,
               Cost max_distance = inf_value()) {
    return run(n, source, -1, std::forward<Expand>(expand), max_distance);
  }

  template<class Expand>
  Cost run_multi(int n, std::span<const std::pair<int,Cost>> sources, int target,
                 Expand&& expand, Cost max_distance = inf_value()) {
    assert(n >= 0 && (target == -1 || (0 <= target && target < n)));
    if (n != n_) reset(n); else { dist_.clear(); parent_.clear(); }
    using Q = std::pair<Cost,int>;
    std::priority_queue<Q, std::vector<Q>, std::greater<Q>> pq;
    for (auto [s,d] : sources) {
      assert(0 <= s && s < n && d >= Cost(0));
      if (d > max_distance) continue;
      if (!dist_.contains(s) || d < dist_.get(s)) {
        dist_.set(s,d); parent_.set(s,-1); pq.push({d,s});
      }
    }
    Cost target_dist = inf_value();
    while (!pq.empty()) {
      auto [du,u] = pq.top(); pq.pop();
      if (!dist_.contains(u) || du != dist_.get(u)) continue;
      if (du > max_distance) break;
      if (u == target) { target_dist = du; break; }
      auto relax = [&](int v, Cost w) {
        assert(0 <= v && v < n && w >= Cost(0));
        if (w > max_distance - du) return;
        Cost nd = du + w;
        if (nd > max_distance) return;
        if (!dist_.contains(v) || nd < dist_.get(v)) {
          dist_.set(v,nd); parent_.set(v,u); pq.push({nd,v});
        }
      };
      expand(u, relax);
    }
    return target == -1 ? Cost(0) : target_dist;
  }

  std::vector<int> path_to(int target) const {
    if (!reached(target)) return {};
    std::vector<int> p;
    for (int v = target; v != -1; v = parent(v)) p.push_back(v);
    std::reverse(p.begin(), p.end());
    return p;
  }
};

} // namespace ahc

// ===== ahc/candidate_graph.hpp =====
#line 1 "ahc/candidate_graph.hpp"
#include <vector>
#include <queue>
#include <algorithm>
#include <cassert>
#include <utility>

namespace ahc {

// Generic bounded candidate graph for routing/local-search neighborhoods.
// Build cost is O(n^2 log k) calls/heap work but memory is O(nk).
class CandidateGraph {
  std::vector<std::vector<int>> g_;
public:
  CandidateGraph() = default;
  explicit CandidateGraph(int n) : g_(n) {}
  void reset(int n) { g_.assign(n, {}); }
  [[nodiscard]] int size() const { return static_cast<int>(g_.size()); }
  const std::vector<int>& neighbors(int u) const { assert(0 <= u && u < size()); return g_[u]; }
  std::vector<int>& neighbors_mut(int u) { assert(0 <= u && u < size()); return g_[u]; }
  bool contains(int u, int v) const {
    const auto& a = neighbors(u);
    return std::find(a.begin(), a.end(), v) != a.end();
  }
  // Deduplicate while preserving candidate priority/order.
  void normalize() {
    StampedArray<unsigned char> seen(size());
    for (auto& a : g_) {
      seen.clear();
      int w = 0;
      for (int v : a) {
        assert(0 <= v && v < size());
        if (!seen.contains(v)) { seen.set(v,1); a[w++] = v; }
      }
      a.resize(w);
    }
  }
  void make_symmetric() {
    std::vector<std::pair<int,int>> add;
    for (int u = 0; u < size(); ++u) for (int v : g_[u]) {
      assert(0 <= v && v < size() && v != u);
      add.push_back({v,u});
    }
    for (auto [u,v] : add) g_[u].push_back(v);
    normalize();
  }

  template<class CostFn>
  static CandidateGraph build_complete(int n, int k, CostFn&& cost,
                                       bool symmetric_union = false) {
    assert(n >= 0 && k >= 0);
    CandidateGraph out(n);
    k = std::min(k, std::max(0, n - 1));
    using Cost = std::decay_t<decltype(cost(0,0))>;
    using Item = std::pair<Cost,int>; // max-heap: worst cost/id on top
    for (int u = 0; u < n; ++u) {
      std::priority_queue<Item> heap;
      for (int v = 0; v < n; ++v) if (v != u) {
        Item x{cost(u,v),v};
        if ((int)heap.size() < k) heap.push(x);
        else if (x < heap.top()) { heap.pop(); heap.push(x); }
      }
      std::vector<Item> tmp; tmp.reserve(heap.size());
      while (!heap.empty()) { tmp.push_back(heap.top()); heap.pop(); }
      std::sort(tmp.begin(), tmp.end());
      auto& a = out.g_[u]; a.reserve(tmp.size());
      for (auto& [c,v] : tmp) a.push_back(v); // nearest/best first
    }
    if (symmetric_union) out.make_symmetric();
    return out;
  }
};

} // namespace ahc

// ===== ahc/route_sequence.hpp =====
#line 1 "ahc/route_sequence.hpp"
#include <type_traits>
#include <utility>
#include <vector>
#include <cassert>

namespace ahc {

// Aggregate for a directed path. fwd is the path cost first->last, rev is the
// cost of traversing the same sequence backwards last->first.
template<class T, class Cost>
struct DirectedPathAggregate {
  bool empty = true;
  int count = 0;
  T first{};
  T last{};
  Cost fwd{};
  Cost rev{};
};

template<class T, class Cost>
struct DirectedPathLift {
  DirectedPathAggregate<T,Cost> operator()(const T& x) const {
    DirectedPathAggregate<T,Cost> a;
    a.empty = false; a.count = 1; a.first = a.last = x;
    a.fwd = a.rev = Cost(0); return a;
  }
};

template<class T, class Cost, class Dist>
struct DirectedPathMerge {
  Dist dist;
  DirectedPathAggregate<T,Cost> operator()(const DirectedPathAggregate<T,Cost>& a,
                                            const DirectedPathAggregate<T,Cost>& b) const {
    if (a.empty) return b;
    if (b.empty) return a;
    DirectedPathAggregate<T,Cost> c;
    c.empty = false; c.count = a.count + b.count;
    c.first = a.first; c.last = b.last;
    c.fwd = a.fwd + dist(a.last,b.first) + b.fwd;
    c.rev = b.rev + dist(b.first,a.last) + a.rev;
    return c;
  }
};

// Directed/asymmetric route sequence on top of AggregatedImplicitSequence.
// Supports reverse/rotate/relocate with exact path/cycle cost in expected O(log n).
template<class T, class Dist,
         class Cost = std::decay_t<decltype(std::declval<Dist>()(
             std::declval<const T&>(), std::declval<const T&>()))>>
class RouteSequence {
  using Agg = DirectedPathAggregate<T,Cost>;
  using Lift = DirectedPathLift<T,Cost>;
  using Merge = DirectedPathMerge<T,Cost,Dist>;
  Dist dist_;
  AggregatedImplicitSequence<T,Agg,Lift,Merge> seq_;
public:
  RouteSequence(const std::vector<T>& route, Dist dist,
                uint64_t seed = 0x9e3779b97f4a7c15ULL)
      : dist_(dist), seq_(route, Agg{}, Lift{}, Merge{dist}, seed) {}
  [[nodiscard]] int size() const { return seq_.size(); }
  [[nodiscard]] bool empty() const { return seq_.empty(); }
  void reverse(int l,int r) { seq_.reverse(l,r); }
  void rotate(int l,int m,int r) { seq_.rotate(l,m,r); }
  void cut_insert(int l,int r,int pos) { seq_.cut_insert(l,r,pos); }
  void insert(int pos,const T& x) { seq_.insert(pos,x); }
  T erase(int pos) { return seq_.erase(pos); }
  void set(int pos,const T& x) { seq_.set(pos,x); }
  std::vector<T> to_vector() { return seq_.to_vector(); }
  Agg aggregate(int l,int r) { return seq_.aggregate(l,r); }
  const Agg& aggregate_all() const { return seq_.aggregate_all(); }
  Cost path_cost() const {
    const Agg& a = seq_.aggregate_all();
    return a.empty ? Cost(0) : a.fwd;
  }
  Cost reverse_path_cost() const {
    const Agg& a = seq_.aggregate_all();
    return a.empty ? Cost(0) : a.rev;
  }
  Cost cycle_cost() const {
    const Agg& a = seq_.aggregate_all();
    if (a.count <= 1) return Cost(0);
    return a.fwd + dist_(a.last,a.first);
  }
};

template<class T, class Dist>
auto make_route_sequence(const std::vector<T>& route, Dist dist,
                         uint64_t seed = 0x9e3779b97f4a7c15ULL) {
  return RouteSequence<T,Dist>(route, std::move(dist), seed);
}

} // namespace ahc

// Reset logical source location for problem-specific code pasted below.
#line 1 "main.cpp"
