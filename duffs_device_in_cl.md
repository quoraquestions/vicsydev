# [vicsy/dev](https://github.com/codr4life/vicsydev) | Duff's Device in Common Lisp
posted Feb 13th 2017, 01:00 am

### preramble
It's a shame that many programming languages fail to provide wholehearted support for green threads, aka. cooperative scheduling; and that fewer still leave enough rope to roll your own. Even Common Lisp fails the test, as it neither provides green threads nor the power needed to implement them; there have been various more or less successful [attempts](https://orthecreedence.github.io/cl-async/2013/03/26/green-threads-and-async-programming.html) over the years, but they all boil down to [pixie dust](http://quickdocs.org/cl-cont/api). One popular urban myth in Common Lisp circles is that green threads bring nothing to the table since threads are already fast enough; which misses the point that cooperative scheduling is a useful, complementary approach to structuring software; as long as performance is at least comparable; and as an added bonus, the performance profile is more consistent and predictable than for preemptive threads.

### Duff's Device in Common Lisp
In C, it's popular to build green threads on top of switches interleaved with user code. The approach doesn't play well with optimising and requires serious hacks to be usable. Implementing the same approach in Lisp takes some imagination, as there are no comparable statements; ```tag-body``` requires clear text labels for jumping, no dynamic look up is possible; and ```case``` doesn't allow fall-through, which is crucial. With jumping off the table, the next possibility is branching around statements; Common Lisp compilers are pretty hard core when it comes to optimisation, which means it's possible to get away with things like this; but modifying unknown Lisp code on statement level is a tar pit.

### Forth
Writing code Forth style allows painless statement level translation because of the simple, linear syntax. That's one of the reasons I chose Forth as the basis of [Lifoo](https://github.com/codr4life/lifoo), a new Forth-based language fused with Common Lisp that I'm working on. By standing on the shoulders of [Lifoo](https://github.com/codr4life/lifoo), we finally get enough leverage to implement the idea in a reasonable amount of reasonable code.

```
Lifoo> 41 1 (1 task-yield +) task 
       run run
       done? swap result swap drop cons

(42 . T)

Lifoo> 42 1 (1 task-yield +) task source

((WHEN (AND (< #:G2537 1) (NOT (TASK-YIELDING? #:G2536)))
          (SETF (TASK-LINE #:G2536) 1)
          (LIFOO-PUSH 1))
        (WHEN (AND (< #:G2537 2) (NOT (TASK-YIELDING? #:G2536)))
          (SETF (TASK-LINE #:G2536) 2)
          (IF (FUNCALL (TYPE-CHECKER [word: task-yield nil]) (STACK *LIFOO*))
              (LIFOO-CALL [word: task-yield nil])
              (LIFOO-CALL 'TASK-YIELD)))
        (WHEN (AND (< #:G2537 3) (NOT (TASK-YIELDING? #:G2536)))
          (SETF (TASK-LINE #:G2536) 3)
          (IF (FUNCALL (TYPE-CHECKER [word: + (number number)])
                       (STACK *LIFOO*))
              (LIFOO-CALL [word: + (number number)])
              (LIFOO-CALL '+))))

Lifoo> 38 
       1 (inc task-yield inc) task drop
       1 (inc task-yield inc) task drop
       finish-tasks drop

42


(defparameter *task* (gensym))

(define-lifoo-init (:task)
  ;; Yields processor to other tasks
  (define-lisp-word :task-yield () ()
    (let ((task (lifoo-var *task*)))
      (assert task)
      (setf (task-yielding? task) t)))

  ;; Pops $code and $num-args,
  ;; and pushes new task
  (define-macro-word :task (in out)
    (declare (ignore in))
    (let ((fst (first out)))
      (cons (cons (first fst)
                  `(lifoo-push (lifoo-task ',(first fst)
                                           (lifoo-pop))))
            (rest out))))

  ;; Runs $1 until next yield or done?
  (define-lisp-word :run (lifoo-task)
      ()
    (lifoo-run-task (lifoo-peek)))

  ;; Runs all tasks that are not done? once
  (define-lisp-word :run-tasks () ()
    (lifoo-push (lifoo-run-tasks)))

  ;; Runs tasks until all are done?
  (define-lisp-word :finish-tasks () (:speed 1)
    (let ((tot 0))
      (do-while ((multiple-value-bind (more? cnt) (lifoo-run-tasks)
                   (incf tot cnt)
                   (> more? 0))))
      (lifoo-push tot)))
  
  ;; Pushes T if $1 is done?,
  ;; NIL otherwise
  (define-lisp-word :done? (lifoo-task) ()
    (lifoo-push (task-done? (lifoo-peek))))

  ;; Pushes result ($1) of $1
  (define-lisp-word :result (lifoo-task) (:speed 1)
    (let* ((task (lifoo-peek))
           (stack (task-stack task)))
      (lifoo-push (lifoo-val (aref stack (1- (length stack))))))))

(defstruct (lifoo-task (:conc-name task-))
  (id (gensym)) stack source fn (line 0 :type fixnum)
  (done? nil :type boolean) (yielding? nil :type boolean))

(defun lifoo-task (code num-args &key (exec *lifoo*))
  "Returns new task for CODE with NUM-ARGS arguments in EXEC"
  (let ((_task (gensym)) (_line (gensym))
        (line 0))
    (declare (type fixnum line))
    (labels ((conv (f)
               (incf line)
               `(when (and (< ,_line ,line)
                           (not (task-yielding? ,_task)))
                  (setf (task-line ,_task) ,line)
                  ,f)))
      (let* ((stack (stack exec))
             (len (length stack))
             (stack2 (subseq stack (- len num-args)))
             (task-stack (make-array (min num-args 3)
                                     :adjustable t
                                     :fill-pointer num-args
                                     :initial-contents stack2))
             (source (mapcar #'conv
                             (lifoo-compile code :exec exec)))
             
             (task
               (make-lifoo-task
                :source source
                :fn (eval `(lambda ()
                             ,(cl4l-optimize)
                             (let* ((,_task (lifoo-var *task*))
                                    (,_line (task-line ,_task)))
                               (declare (ignorable ,_task ,_line))
                               ,@source)))
                :stack task-stack)))
        (push task (lifoo-var *tasks* :exec exec))
        task))))

(defun lifoo-run-task (task &key (exec *lifoo*))
  "Runs TASK once in EXEC"
  (assert (not (task-done? task)))
  (let ((prev (stack exec)))
    (setf (lifoo-var *task* :exec exec) task)
    (setf (stack exec) (task-stack task))
    (unwind-protect
         (funcall (task-fn task))
      (setf (stack exec) prev)
      (setf (lifoo-var *task* :exec exec) nil)))
  (if (task-yielding? task)
      (setf (task-yielding? task) nil)
      (setf (task-done? task) t)))

(defun lifoo-run-tasks (&key (exec *lifoo*))
  "Runs all tasks in EXEC that are not DONE? once"
  (let ((rem 0) (cnt 0))
    (labels ((rec (in out)
               (if in
                   (let ((task (first in)))
                     (lifoo-run-task task :exec exec)
                     (incf cnt)
                     (rec (rest in)
                          (if (task-done? task)
                              out
                              (progn
                                (incf rem)
                                (cons task out)))))
                   (nreverse out))))
      (setf (lifoo-var *tasks* :exec exec)
            (rec (lifoo-var *tasks*) nil)))
    (values rem cnt)))
```

### semantics
One of the drawbacks of this approach is that yields are only allowed from task scope, yields from nested scopes are obeyed on scope exit. I'm still marinating a few different approaches for dealing with that issue. But the added flexibility when it comes to task management still makes it a usable tool.

### performance
As of right now, threads and cooperative tasks are comparable in Lifoo when it comes to performance; but there is plenty of more low hanging optimisation fruit left in the task code path. Each repetition is further repeated x 30, meaning 300 runs in this example. ```cl4l:*cl4l-speed*``` may be set to a value between 1 and 3 to optimize most of the code involved in one go.

```
CL-USER> (cl4l-test:run-suite '(:lifoo :task :perf) :reps 10)
(lifoo task perf)              2.94
(lifoo task spawn perf)       3.172
TOTAL                         6.112


(define-test (:lifoo :task :perf)
  (lifoo-asseq 0
    0
    1 (1 asseq dec task-yield
       1 asseq dec task-yield
       1 asseq dec) task drop
    1 (0 asseq inc task-yield
       0 asseq inc task-yield
       0 asseq inc) task drop
    finish-tasks drop))

(define-test (:lifoo :task :spawn :perf)
  (lifoo-asseq 0
    0 chan
    1 (1 send 2 send 3 send)@ spawn swap
    1 (recv 1 asseq drop
       recv 2 asseq drop
       recv 3 asseq drop)@ spawn
    wait drop swap wait length))
```

You may find more in the same spirit [here](http://vicsydev.blogspot.de/) and [here](https://github.com/codr4life/vicsydev), and a full implementation of this idea and more [here](https://github.com/codr4life).

peace, out