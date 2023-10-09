//Qt\6.4.2\Src\qtbase\src\corelib\kernel\qobject.cpp
template <bool callbacks_enabled>
void doActivate(QObject *sender, int signal_index, void **argv)
{
    // 获取信号发送者的指针,  
	// static QObjectPrivate *get(QObject *o) { return o->d_func(); }
    // static const QObjectPrivate *get(const QObject *o) { return o->d_func(); }
    QObjectPrivate *sp = QObjectPrivate::get(sender);
    // QObjectPrivate的blockSig属性,  默认为false, 表示not blocking signals
	// 如果该标志位为true, 则表示blocking siganls, 那么槽函数不会被调用
    if (sp->blockSig)
        return;
    // Q_TRACE_SCOPE(tracepoint, args...): 用于创建追踪作用域，包括进入和退出。
    // 它首先触发 ${tracepoint}_entry，然后在作用域退出时触发 ${tracepoint}_exit
    Q_TRACE_SCOPE(QMetaObject_activate, sender, signal_index);
    // 声明式信号连接检查，一个指向QObjectPrivate对象且名为sp的指针是否与一个特定的声明式信号通过signal_index索引连接。
	// 这是一个条件语句，只有当信号已经连接时才会执行以下代码块
    if (sp->isDeclarativeSignalConnected(signal_index)
            && QAbstractDeclarativeData::signalEmitted) {
		// 使用Q_TRACE_SCOPE宏来启用一个追踪作用域，用于跟踪声明式信号的激活。
		// Q_TRACE_SCOPE通常用于性能分析和调试，可以跟踪代码的执行时间等信息。
        Q_TRACE_SCOPE(QMetaObject_activate_declarative_signal, sender, signal_index);
		// 处理声明式信号的激活
        QAbstractDeclarativeData::signalEmitted(sp->declarativeData, sender,
                                                signal_index, argv);
    }
	//qt_signal_spy_callback_set 是一个 QBasicAtomicPointer 类型的全局变量，用于存储信号监视器的回调函数集合
	//【qobject.cpp 3875行】Q_CORE_EXPORT QBasicAtomicPointer<QSignalSpyCallbackSet> qt_signal_spy_callback_set 
	// = Q_BASIC_ATOMIC_INITIALIZER(nullptr) 
    //如果callbacks_enabled为true则将指向QSignalSpyCallbackSet的指针赋值给signal_spy_set
	//loadAcquire()是一个原子操作，用于获取变量的当前值
    const QSignalSpyCallbackSet *signal_spy_set = callbacks_enabled ? qt_signal_spy_callback_set.loadAcquire() : nullptr;

    void *empty_argv[] = { nullptr };
    if (!argv)
        argv = empty_argv;
    // 是否存在潜在的声明式连接, 如果不存在则直接return
    if (!sp->maybeSignalConnected(signal_index)) {
        // The possible declarative connection is done, and nothing else is connected
		// 检查 callbacks_enabled 是否为真，以及是否存在名为 signal_begin_callback 的回调函数。
		// 如果这两个条件都满足，将调用 signal_spy_set->signal_begin_callback，这个回调函数通常用于跟踪信号的开始
        if (callbacks_enabled && signal_spy_set->signal_begin_callback != nullptr)
            signal_spy_set->signal_begin_callback(sender, signal_index, argv);
		// 检查 callbacks_enabled 是否为真，以及是否存在名为 signal_end_callback 的回调函数。
		// 如果这两个条件都满足，将调用 signal_spy_set->signal_end_callback，这个回调函数通常用于跟踪信号的结束
        if (callbacks_enabled && signal_spy_set->signal_end_callback != nullptr)
            signal_spy_set->signal_end_callback(sender, signal_index);
        return;
    }
	// 检查 callbacks_enabled 是否为真，以及是否存在名为 signal_begin_callback 的回调函数。
	// 如果这两个条件都满足，将调用 signal_spy_set->signal_begin_callback，这个回调函数通常用于跟踪信号的开始
    if (callbacks_enabled && signal_spy_set->signal_begin_callback != nullptr)
        signal_spy_set->signal_begin_callback(sender, signal_index, argv);

    bool senderDeleted = false;
    {
    Q_ASSERT(sp->connections.loadAcquire());
	// ConnectionDataPointer是一个指向 ConnectionData的指针，用于跟踪和管理连接信息，
	// ConnectionData结构包含了一个 SignalVector指针，用于存储与对象关联的信号连接信息
    QObjectPrivate::ConnectionDataPointer connections(sp->connections.loadRelaxed());
	// SignalVector指针，用于存储与对象关联的信号连接信息
    QObjectPrivate::SignalVector *signalVector = connections->signalVector.loadRelaxed();
    // 从signalVector里面按singal_index取出某个ConnectionList(某个信号的所有连接信息，一个信号可以连接到多个槽函数，
	// 所以对每个信号都可能有多个连接)
    const QObjectPrivate::ConnectionList *list;
    if (signal_index < signalVector->count())
        list = &signalVector->at(signal_index);
    else
        list = &signalVector->at(-1);
    //获取当前线程
    Qt::HANDLE currentThreadId = QThread::currentThreadId();
	//判断当前线程是否与发送信号的QObject对象的线程相同
    bool inSenderThread = currentThreadId == QObjectPrivate::get(sender)->threadData.loadRelaxed()->threadId.loadRelaxed();

    // We need to check against the highest connection id to ensure that signals added
    // during the signal emission are not emitted in this emission.
	//在信号发射期间，需要检查最高的连接 ID（Connection ID），以确保在当前信号发射过程中添加的信号不会在同一信号发射中再次触发。
    //在 Qt 的信号和槽机制中，一个 QObject 对象可以连接到其他 QObject 对象的信号。
	//每个连接都会分配一个唯一的连接 ID，用于标识连接。信号的发射是通过调用与之连接的槽函数来触发的。
    //上述引用的文本提到的情况是在信号发射期间，可能会在发射过程中添加新的信号连接。
	//为了避免出现问题，需要确保新连接的信号不会在当前信号发射中再次触发，因此需要检查并保持对最高连接 ID 的跟踪。
	//这可以通过比较新连接的连接 ID 与当前信号发射过程中的最高连接 ID 来实现。
	//这个检查的目的是确保信号发射过程中的稳定性和一致性，以避免潜在的问题，如无限递归或不明确的信号触发。
	//所以，在添加新的信号连接时，需要谨慎考虑当前的信号发射上下文，以避免不必要的复杂性。
	
	//获取当前对象连接的最高ID
    uint highestConnectionId = connections->currentConnectionId.loadRelaxed();
	//遍历信号signal_index的所有连接，一个信号对应多个槽函数，所以每个signal_index对应多个连接
    do {
		//这段代码的目的是在连接列表中迭代查找连接，并在找到第一个连接后，检查是否存在。
		//如果存在连接，代码将继续执行后续操作；
		//如果没有找到连接，代码将跳过当前循环迭代，继续查找下一个连接
        QObjectPrivate::Connection *c = list->first.loadRelaxed();
        if (!c)
            continue;

        do {
			//首先，从 QObjectPrivate::Connection结构中加载与当前连接相关的接收者对象的指针。
			//这个接收者对象是连接的目标，也就是信号要传递到的对象
            QObject * const receiver = c->receiver.loadRelaxed();
			//检查接收者对象是否存在。如果接收者对象不存在(可能已被析构或无效)，则跳过当前连接，继续处理下一个连接
            if (!receiver)
                continue;
            //从连接中加载与接收者对象关联的线程数据(QThreadData)。Qt使用线程数据(QThreadData)来跟踪对象所在的线程等信息
            QThreadData *td = c->receiverThreadData.loadRelaxed();
			//检查线程数据是否存在。如果线程数据不存在，表示接收者对象可能已经被析构或无效，继续处理下一个连接。
            if (!td)
                continue;
            //定义一个布尔变量 receiverInSameThread，用于标识信号发送者和接收者是否位于同一线程中。
            bool receiverInSameThread;
			/*inSenderThread的值表示当前线程是否与发送者(发送信号的QObject对象)的线程相同。
			如果inSenderThread为true，则表示当前线程与发送者的线程一致，
			那么比较当前线程ID和接收者线程ID，就能确定信号发送者与接收者是否在同一线程中。
			如果inSenderThread为 false，则表示当前线程与发送者的线程不一致，
			那么在比较当前线程ID和接收者线程ID之前，需要获取接收者对象的互斥锁(signalSlotLock(receiver))
			来防止与moveToThread操作的干扰。
			最终，receiverInSameThread标志用于确定是否可以立即发送信号，还是需要将信号排入事件队列中
			receiverInSameThread标志为true表示可以立即发送信号，
			receiverInSameThread标志为false表示需要将信号排入事件队列中(不可以立即发送信号)
			*/
            if (inSenderThread) {
                receiverInSameThread = currentThreadId == td->threadId.loadRelaxed();
            } else {
                // need to lock before reading the threadId, because moveToThread() could interfere
                QMutexLocker lock(signalSlotLock(receiver));
                receiverInSameThread = currentThreadId == td->threadId.loadRelaxed();
            }
		   // determine if this connection should be sent immediately or
           // put into the event queue
		   //如果connectionType是Qt::AutoConnection并且receiverInSameThread为False
		   //或者connectionType是Qt::QueuedConnection, 则进行队列处理
            if ((c->connectionType == Qt::AutoConnection && !receiverInSameThread)
                || (c->connectionType == Qt::QueuedConnection)) {
                queued_activate(sender, signal_index, c, argv);
                continue;
#if QT_CONFIG(thread)
            } 
			//如果connectionType是Qt::BlockingQueuedConnection，则进行阻塞处理
			else if (c->connectionType == Qt::BlockingQueuedConnection) {
			    //检查信号的发送者和接收者是否在同一个线程中。
			   //如果它们在同一个线程中，那么就会输出一条警告信息，
			   //因为在同一个线程中使用BlockingQueuedConnection可能会导致死锁问题。
                if (receiverInSameThread) {
                    qWarning("Qt: Dead lock detected while activating a BlockingQueuedConnection: "
                    "Sender is %s(%p), receiver is %s(%p)",
                    sender->metaObject()->className(), sender,
                    receiver->metaObject()->className(), receiver);
                }
				/*一次性连接(Single-Shot Connection)
				isSingleShot是struct QObjectPrivate::Connection结构体中的一个ushort类型的成员变量，
				用来表示信号槽连接是否是一次性连接。
				如果它的值为1，则表示这个连接是一次性连接；如果值为0，则表示不是一次性连接。
                一次性连接(Single-Shot Connection)是一种特殊类型的信号槽连接，
				它的特点是只会在第一次触发信号时执行槽函数，之后会自动断开连接。
				在 Qt 中，一次性连接可以通过QObject::connect函数的Qt::UniqueConnection标志来创建，
			    也可以通过QTimer::singleShot静态函数创建。
				检查连接是否为一次性连接，并且尝试从连接列表中移除连接。
				如果成功移除连接，那么执行后续代码，否则continue继续下一个连接
				*/
                if (c->isSingleShot && !QObjectPrivate::removeConnection(c))
                    continue;

                QSemaphore semaphore;
                {
                    QBasicMutexLocker locker(signalSlotLock(receiver));
                    if (!c->isSingleShot && !c->receiver.loadAcquire())
                        continue;
                    QMetaCallEvent *ev = c->isSlotObject ?
                        new QMetaCallEvent(c->slotObj, sender, signal_index, argv, &semaphore) :
                        new QMetaCallEvent(c->method_offset, c->method_relative, c->callFunction,
                                           sender, signal_index, argv, &semaphore);
                    QCoreApplication::postEvent(receiver, ev);
                }
                semaphore.acquire();
                continue;
#endif
            }
            //检查连接是否为一次性连接，并且尝试从连接列表中移除连接。
		    //如果成功移除连接，那么执行后续代码，否则continue继续下一个连接
            if (c->isSingleShot && !QObjectPrivate::removeConnection(c))
                continue;
			/* QObjectPrivate::Sender 
			当信号被发射时，这个结构会被传递给与信号关联的槽函数，
			以确保槽函数能够了解信号的发送者、接收者和信号索引等上下文信息。
            创建一个senderData对象，它包含了发送者、接收者以及信号索引等信息，用于后续的槽函数调用。
			(1) 第一个参数，是一个条件表达式，receiverInSameThread ? receiver : nullptr：
			如果receiverInSameThread为真，那么它将被设置为receiver，否则为nullptr。
			表示指向信号接收者对象的指针。
			(2) 第二个参数，sender, 表示指向信号发送者对象的指针。
            (3) 第三个参数，signal_index, 它是一个表示信号索引的整数, 每个信号都在
			元对象(meta-object)中具有一个唯一的索引，用于标识信号的类型
			*/
            QObjectPrivate::Sender senderData(receiverInSameThread ? receiver : nullptr, sender, signal_index);
            /*
			第一个分支处理连接是槽对象(SlotObject)的情况
			这个条件检查连接是否为SlotObject。
			如果是，那么它使用Q_TRACE_SCOPE宏来跟踪SlotObject的调用，
			并使用obj->call(receiver, argv)调用槽对象的call方法来执行槽函数
			*/
            if (c->isSlotObject) {
                SlotObjectGuard obj{c->slotObj};
                {
                    Q_TRACE_SCOPE(QMetaObject_activate_slot_functor, c->slotObj);
                    obj->call(receiver, argv);
                }
            } 
			/*
			第二个分支处理连接是普通槽函数的情况。
			如果连接不是SlotObject，并且条件满足，它会执行这个分支。
			它首先检查普通槽函数是否存在(c->callFunction)，
			以及普通槽函数的索引是否在接收者的元对象中
			(c->method_offset <= receiver->metaObject()->methodOffset())。
			如果满足条件，它会跟踪普通槽函数的开始(如果启用了信号跟踪)，
			然后调用callFunction来执行普通槽函数
			*/
			else if (c->callFunction && c->method_offset <= receiver->metaObject()->methodOffset()) {
                //we compare the vtable to make sure we are not in the destructor of the object.
				//创建了一个名为 method_relative 的整数变量，用于存储连接的普通槽函数在方法表中的相对偏移量
                const int method_relative = c->method_relative;
				//创建了一个自动类型推断的变量 callFunction，用于存储连接的普通槽函数的函数指针。
				//这个函数指针将在后续用于调用槽函数。
                const auto callFunction = c->callFunction;
				//这里创建了一个整数变量methodIndex，用于存储连接的普通槽函数在方法表中的索引位置。
				//如果Qt配置中启用了跟踪点(tracepoints)或callbacks_enabled为true，
				//则使用c->method()获取索引值，否则设为0
                const int methodIndex = (Q_HAS_TRACEPOINTS || callbacks_enabled) ? c->method() : 0;
				//这是一个条件语句，用于检查callbacks_enabled是否为true并且slot_begin_callback不为空。
				//如果条件满足，表示可以调用slot_begin_callback。
                if (callbacks_enabled && signal_spy_set->slot_begin_callback != nullptr)
                    signal_spy_set->slot_begin_callback(receiver, methodIndex, argv);
                {
					//这是一个宏，用于跟踪槽函数的调用，记录一些相关信息。
                    Q_TRACE_SCOPE(QMetaObject_activate_slot, receiver, methodIndex);
					//这里调用了槽函数，通过callFunction函数指针调用，
					//传递了接收信号的对象receiver、调用类型QMetaObject::InvokeMetaMethod、
					//槽函数的相对偏移量method_relative 和参数数组argv。
                    callFunction(receiver, QMetaObject::InvokeMetaMethod, method_relative, argv);
                }
                //这是一个条件语句，用于检查callbacks_enabled是否为true并且slot_end_callback不为空。
				//如果条件满足，表示可以调用slot_end_callback。
                if (callbacks_enabled && signal_spy_set->slot_end_callback != nullptr)
                    signal_spy_set->slot_end_callback(receiver, methodIndex);
            } 
			/*
			第三个分支处理其它情况，即连接到一个不符合上述两种情况的槽函数上。
			这些情况可能是Lambda表达式或全局函数等。
			如果连接既不是SlotObject，也不满足上述条件，那么它会执行这个分支。
			这个分支用于处理通过元对象系统调用槽函数的情况。
			它跟踪槽函数的开始(如果启用了信号跟踪)，然后调用QMetaObject::metacall来执行槽函数。
			*/
			else {
				//这一行计算要调用的槽函数的索引。
				//method_offset表示当前槽函数在接收对象的方法表中的偏移量，
				//而method_relative表示这个槽函数在整个Qt应用程序的方法表中的位置。
				//这两个值的和就是要调用的槽函数在接收对象的方法表中的索引
                const int method = c->method_relative + c->method_offset;
                //这一行检查callbacks_enabled是否为true，并且slot_begin_callback不为空。
                //如果满足条件，它会调用slot_begin_callback，用于记录或跟踪槽函数的开始执行。
                if (callbacks_enabled && signal_spy_set->slot_begin_callback != nullptr) {
                    signal_spy_set->slot_begin_callback(receiver, method, argv);
                }
                //这是实际调用槽函数的部分。QMetaObject::metacall是Qt元对象系统的一部分，
				//它用于动态调用槽函数。它会根据给定的索引(method)在接收对象上调用相应的槽函数，
				//传递给槽函数的参数存储在argv中。QMetaObject::InvokeMetaMethod表示这是一个元对象方法的调用。
                {
                    Q_TRACE_SCOPE(QMetaObject_activate_slot, receiver, method);
                    QMetaObject::metacall(receiver, QMetaObject::InvokeMetaMethod, method, argv);
                }
                //这一行检查callbacks_enabled是否为true，并且slot_end_callback不为空。
                //如果满足条件，它会调用slot_end_callback，用于记录或跟踪槽函数的结束执行。
                if (callbacks_enabled && signal_spy_set->slot_end_callback != nullptr)
                    signal_spy_set->slot_end_callback(receiver, method);
            }
        } 
		//这一行代码从当前连接c的nextConnectionList中加载下一个连接。
		//nextConnectionList是指向下一个连接的指针。
		//loadRelaxed()是一种加载操作，用于获取指针的值，不会引入额外的同步或内存顺序要求
		//这是另一个条件检查，用于确保连接的id(连接的唯一标识符)小于或等于highestConnectionId。
		//这个条件可能用于控制循环的范围，只处理特定范围内的连接。
		while ((c = c->nextConnectionList.loadRelaxed()) != nullptr && c->id <= highestConnectionId);

    }
	//list != &signalVector->at(-1)是外层循环的循环结束语句。
	//list = &signalVector->at(-1))的目的是在每次迭代时，将list指针重置为指向signalVector 中的最后一个元素。
	while (list != &signalVector->at(-1) &&
        //start over for all signals;
        ((list = &signalVector->at(-1)), true));

        //这行代码检查connections中的currentConnectionId是否等于0。
		//currentConnectionId是一个标识连接的计数器或标识符。
		//如果它等于0，表示没有活动的连接，可能意味着发送者已经被删除。
		//如果条件成立，那么设置senderDeleted为true，表示发送者已经被删除
        if (connections->currentConnectionId.loadRelaxed() == 0)
            senderDeleted = true;
    }
	//这是一个条件检查，确保 senderDeleted为false。
	//如果senderDeleted为true，则说明发送者已被删除，不需要进一步处理
    if (!senderDeleted) {
		//这一行代码将调用cleanOrphanedConnections函数。
		//这个函数用于清理与信号发送者关联的孤立连接，即那些已经不再有效的连接。
		//这可以防止悬挂引用和资源泄漏。
        sp->connections.loadRelaxed()->cleanOrphanedConnections(sender);
        //这是一个条件检查，确保callbacks_enabled为true并且signal_end_callback不为空。
		//如果这两个条件都满足，那么它表示用户可能已经设置了signal_end_callback
        if (callbacks_enabled && signal_spy_set->signal_end_callback != nullptr)
			//如果前面的条件都成立，那么就会调用signal_end_callback函数，
		    //传递了发送者sender和信号的索引signal_index作为参数。
            signal_spy_set->signal_end_callback(sender, signal_index);
    }
}
