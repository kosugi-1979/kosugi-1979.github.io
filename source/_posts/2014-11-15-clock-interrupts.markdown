---
layout: post
title: "FreeBSD/amd64 の clock 割り込み"
date: 2014-11-15 20:16:42 +0900
comments: true
categories: FreeBSD
---
	% uname -a
	FreeBSD freebsd11 11.0-CURRENT FreeBSD 11.0-CURRENT #3 r272236: Sat Nov  1 17:27:44 JST 2014     root@freebsd11:/usr/obj/usr/src/sys/GENERIC  amd64

"The Design and Implementation of the FreeBSD Operating System Second Edition" には clock ticks に対する割り込みは hardclock() を呼ぶと書かれているのだけれど、どうも実際には hardclock() は呼ばれず、代わりに kern_clocksource.c 内の handleevents() とそこから呼ばれている kern_clock.c 内のhardclock_cnt() で本に書かれている処理が行われているようだ。

hardclock() で行われるとされているのは次の通り。

- 実行中のプロセスが virtual または profiling interval timer を持っているならタイマをデクリメントし、時間が来たら signal を発行
- 前の hardclock() からの tick 数だけ現在時刻を更新
- softclock() を走らせる必要があるなら softclock プロセスを 実行可能にする

この他、別の profile 用の clock がなければ、 profclock() でやることを hardclock() 内でやるとのこと。

## 実行中のプロセスが virtual または profiling interval timer を持っているならタイマをデクリメントし、時間が来たら signal を発行

これは hardclock_cnt() にある。

	if (usermode &&
	    timevalisset(&pstats->p_timer[ITIMER_VIRTUAL].it_value)) {
		PROC_SLOCK(p);
		if (itimerdecr(&pstats->p_timer[ITIMER_VIRTUAL],
		    tick * cnt) == 0)
			flags |= TDF_ALRMPEND | TDF_ASTPENDING;
		PROC_SUNLOCK(p);
	}
	if (timevalisset(&pstats->p_timer[ITIMER_PROF].it_value)) {
		PROC_SLOCK(p);
		if (itimerdecr(&pstats->p_timer[ITIMER_PROF],
		    tick * cnt) == 0)
			flags |= TDF_PROFPEND | TDF_ASTPENDING;
		PROC_SUNLOCK(p);
	}
	thread_lock(td);
	sched_tick(cnt);
	td->td_flags |= flags;
	thread_unlock(td);

本の内容そのままといった感じ。ただ、その場で signal を発行するのではなく、ここでは thread structure flag のフラグをセットするだけのようだ。また、virtual interval timer に関してはユーザモードの時のみ処理を行う。

## 前の hardclock() からの tick 数だけ現在時刻を更新

これは hardclock_cnt() から呼ばれる tc_ticktock()、tc_windup() で行われているようだ。tc_windup() には以下のような処理がある。

        th->th_offset_count += delta;
        th->th_offset_count &= th->th_counter->tc_counter_mask;
        while (delta > th->th_counter->tc_frequency) {
                /* Eat complete unadjusted seconds. */
                delta -= th->th_counter->tc_frequency;
                th->th_offset.sec++;
        }
        if ((delta > th->th_counter->tc_frequency / 2) &&
            (th->th_scale * delta < ((uint64_t)1 << 63))) {
                /* The product th_scale * delta just barely overflows. */
                th->th_offset.sec++;
        }
        bintime_addx(&th->th_offset, th->th_scale * delta);

delta は ACPI の  High Precision Event Timer を使って得た前回の tc_windup() 呼出しからの差分。この値を用いて th(timehands) を更新している。なお、gettimeofday() などではこの timehands を参照する。

## softclock() を走らせる必要があるなら softclock プロセスを 実行可能にする

softclock() の必要性の判定は handleevents() 内で次のようになっている。

	if (now >= state->nextcallopt) {
		state->nextcall = state->nextcallopt = SBT_MAX;
		callout_process(now);
	}

現在時刻が state->nextcallopt よりも先に進んでいたら callout_process() を呼ぶ。callout_process() で softclock に関する部分は、少し長いが以下のようになっている。

		/* Iterate callwheel from firstb to nowb and then up to lastb. */
		do {
			sc = &cc->cc_callwheel[firstb & callwheelmask];
			tmp = LIST_FIRST(sc);
			while (tmp != NULL) {
				/* Run the callout if present time within allowed. */
				if (tmp->c_time <= now) {
					/*
					 * Consumer told us the callout may be run
					 * directly from hardware interrupt context.
					 */
					if (tmp->c_flags & CALLOUT_DIRECT) {
	#ifdef CALLOUT_PROFILING
						++depth_dir;
	#endif
						cc->cc_exec_next_dir =
						    LIST_NEXT(tmp, c_links.le);
						cc->cc_bucket = firstb & callwheelmask;
						LIST_REMOVE(tmp, c_links.le);
						softclock_call_cc(tmp, cc,
	#ifdef CALLOUT_PROFILING
						    &mpcalls_dir, &lockcalls_dir, NULL,
	#endif
						    1);
						tmp = cc->cc_exec_next_dir;
					} else {
						tmpn = LIST_NEXT(tmp, c_links.le);
						LIST_REMOVE(tmp, c_links.le);
						TAILQ_INSERT_TAIL(&cc->cc_expireq,
						    tmp, c_links.tqe);
						tmp->c_flags |= CALLOUT_PROCESSED;
						tmp = tmpn;
					}
					continue;
				}
				/* Skip events from distant future. */
				if (tmp->c_time >= max)
					goto next;
				/*
				 * Event minimal time is bigger than present maximal
				 * time, so it cannot be aggregated.
				 */
				if (tmp->c_time > last) {
					lastb = nowb;
					goto next;
				}
				/* Update first and last time, respecting this event. */
				if (tmp->c_time < first)
					first = tmp->c_time;
				tmp_max = tmp->c_time + tmp->c_precision;
				if (tmp_max < last)
					last = tmp_max;
	next:
				tmp = LIST_NEXT(tmp, c_links.le);
			}
			/* Proceed with the next bucket. */
			firstb++;
			/*
			 * Stop if we looked after present time and found
			 * some event we can't execute at now.
			 * Stop if we looked far enough into the future.
			 */
		} while (((int)(firstb - lastb)) <= 0);

一番外側の do-while 文は大雑把には

		/* Iterate callwheel from firstb to nowb and then up to lastb. */
		do {
			sc = &cc->cc_callwheel[firstb & callwheelmask];

			/* Proceed with the next bucket. */
			firstb++;
		} while (((int)(firstb - lastb)) <= 0);

となっているので、冒頭のコメントの通り firstb から lastb まで callwheel を見ていく処理になっている。内側にはもう一つ while 文があり、そちらは

			tmp = LIST_FIRST(sc);
			while (tmp != NULL) {
				/* Run the callout if present time within allowed. */
	next:
				tmp = LIST_NEXT(tmp, c_links.le);
			}

となっていることから、callwheel は callwheelmask + 1 個のエントリを持つ ring buffer で、各エントリがリストになっているデータ構造だとわかる。上述の二重ループは firstb から lastb までの各エントリに対して処理を行うためのループだということ。ここまで来たらわかると思うが、この callwheel は本でいうところの callout queue そのもの。各エントリが持つ c_time と現在時刻 now を比べて、c_time が now よりも後なら、そのエントリに対応する処理をスケジュールしているのが次の部分。

				/* Run the callout if present time within allowed. */
				if (tmp->c_time <= now) {
					/*
					 * Consumer told us the callout may be run
					 * directly from hardware interrupt context.
					 */
					if (tmp->c_flags & CALLOUT_DIRECT) {
	#ifdef CALLOUT_PROFILING
						++depth_dir;
	#endif
						cc->cc_exec_next_dir =
						    LIST_NEXT(tmp, c_links.le);
						cc->cc_bucket = firstb & callwheelmask;
						LIST_REMOVE(tmp, c_links.le);
						softclock_call_cc(tmp, cc,
	#ifdef CALLOUT_PROFILING
						    &mpcalls_dir, &lockcalls_dir, NULL,
	#endif
						    1);
						tmp = cc->cc_exec_next_dir;
					} else {
						tmpn = LIST_NEXT(tmp, c_links.le);
						LIST_REMOVE(tmp, c_links.le);
						TAILQ_INSERT_TAIL(&cc->cc_expireq,
						    tmp, c_links.tqe);
						tmp->c_flags |= CALLOUT_PROCESSED;
						tmp = tmpn;
					}
					continue;
				}

見れば分かるが、 CALLOUT_DIRECT フラグが立っている場合には現在の割り込み処理の中で直接 softclock_call_cc() を呼んでいる。立っていなければ、expireq に enqueue する。enqueue したエントリは softclock() で処理される。

で、肝心の softclock() のスケジュールは callout_process() の最後、

        if (!TAILQ_EMPTY(&cc->cc_expireq))
                swi_sched(cc->cc_cookie, 0);

で行う。cc_cookie が interrupt handler で、kern_timeout.c の start_softclock() で softclock() が呼ばれるように初期化している。
