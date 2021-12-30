---
layout: post
title: How to add a background thread to a Flask app?
tags: [python, threading, flask]
comments: false
readtime: true
slug: python-flask-background-thread
---

This short article covers a template for setting up a Flask based web app 
with a background thread. 
Knowledge of Python and basic knowledge of Flask is required. 

## Introduction

Imagine the situation where we have a web app that accepts some requests via HTTP.
These requests can generate longer-running tasks. 
We do not want the user to wait until long-running tasks are finished to receive a response 
and, also, want to keep our app as simple as possible. 
So, setting up dedicated software like [Celery](https://docs.celeryproject.org/en/stable/) is not an option.
One possible solution is to set up a separate *background Python thread* alongside the main thread on which the Flask app is running. 
Let's call this background thread a **scheduler**.

## Setting up a background scheduler

The Flask-based web app will handle new tasks via HTTP POST requests.
The scheduler will run in a **separate thread**. 
The app and scheduler will communicate via the Python `queue.Queue` object 
representing a simple thread-safe queue with put/get operations.
New tasks will be put to the queue by the Flask app, 
and the scheduler will get them from the queue and process.
When a program terminates, the scheduler cleans up its resources before shutdown
(so-called *graceful shutdown*).

{: .box-warning}
**Warning:** In case of an app restart, items in the queue will be lost. 
This is *simple* implementation. 
If you have critical data there, consider using dedicated software like Celery. 

Our sample project contains two files:

- `app_factory.py`, to configure Flask app
- `scheduler.py`, where the scheduler code is implemented


Let's take a look at `app_factory.py`:

```python
import logging
import signal

from flask import Flask, request, jsonify
from scheduler import TASKS_QUEUE, SchedulerFactory


def create_app():
    # for development purposes only
    logging.basicConfig(level=logging.DEBUG)

    app = Flask(__name__)

    @app.route('/task', methods=['POST'])
    def submit_task():
        task = request.json
        logging.debug(f'Received task via POST: {task}')

        TASKS_QUEUE.put(task)
        return jsonify({'success': 'OK'})

    scheduler = SchedulerFactory.create_scheduler()
    scheduler.start()

    # it is responsibility of a caller to stop a scheduler
    # when main thread exits
    def sigint_handler(signum, frame):
        scheduler.stop()
        exit(0)
    
    signal.signal(signal.SIGINT, sigint_handler)

    return app
```

This is a straightforward setup of a Flask app 
([tutorial to setup](https://flask.palletsprojects.com/en/2.0.x/tutorial/factory/)). 
`TASKS_QUEUE` is a `queue.Queue` object 
and is shared between `submit_task` handler and the scheduler.

`SchedulerFactory` is used to create an instance of a `Scheduler` class.
`scheduler.start()` launches the scheduler in a separate thread.

When a program is terminated, the `SIGINT` signal is received,
and, using `signal` Python module, custom handler, `sigint_handler`, is executed.
`sigint_handler` stops the scheduler by calling `scheduler.stop()` and exits the program.

Now let's take a look at `scheduler.py`, where `Scheduler` is implemented:

```python
import logging
import queue
import threading
import time
from queue import Queue
from abc import abstractmethod, ABC
from typing import Dict

TASKS_QUEUE = Queue()


class Scheduler(threading.Thread, ABC):
    def __init__(self):
        super(Scheduler, self).__init__()
        self._stop_event = threading.Event()

    def stop(self) -> None:
        self._stop_event.set()

    def _stopped(self) -> bool:
        return self._stop_event.is_set()

    @abstractmethod
    def startup(self) -> None:
        """
        Method that is called before the scheduler starts.
        Init all necessary resources here.
        :return: None
        """
        raise NotImplemented

    @abstractmethod
    def shutdown(self) -> None:
        """
        Method that is called shortly after stop() method was called.
        Use it to clean up all resources before scheduler exits.
        :return: None
        """
        raise NotImplemented

    @abstractmethod
    def handle(self) -> None:
        """
        Method that should contain business logic of the scheduler.
        Will be executed in the loop until stop() method is called.
        Must not block for a long time.
        :return: None
        """
        raise NotImplemented

    def run(self) -> None:
        """
        This method will be executed in a separate thread
        when Scheduler.start() is called.
        :return: None
        """
        self.startup()
        while not self._stopped():
            self.handle()
        self.shutdown()


class DummyScheduler(Scheduler):
    def startup(self) -> None:
        logging.info('Dummy scheduler starts')

    def shutdown(self) -> None:
        logging.info('Dummy scheduler stops')

    def handle(self) -> None:
        try:
            TASKS_QUEUE.get(block=False)
            logging.info(f'Scheduler received a task. Nothing to do.')
        except queue.Empty:
            time.sleep(1)


class FifoScheduler(Scheduler):
    def startup(self) -> None:
        logging.info('FIFO scheduler starts')

    def shutdown(self) -> None:
        logging.info('FIFO scheduler stops')

    @staticmethod
    def _process_task(task: Dict) -> None:
        logging.info(f'Scheduler processing task: {task}')

    def handle(self) -> None:
        try:
            task = TASKS_QUEUE.get(block=False)
            logging.info(f'Scheduler received a task')
            self._process_task(task)
        except queue.Empty:
            time.sleep(1)


class SchedulerFactory:
    @staticmethod
    def create_scheduler(scheduler_type: str = 'fifo') -> Scheduler:
        if scheduler_type == 'fifo':
            return FifoScheduler()
        # new types of schedulers can be added like that
        # if scheduler_type == 'cool_scheduler':
        #     return CoolScheduler()
        return DummyScheduler()
```

All concrete schedulers like `FifoScheduler` or `DummyScheduler`
inherit from the base abstract class `Scheduler`.

`Scheduler` inherits from the `threading.Thread` class to simplify 
operations on a thread. 
In addition, it also implements the mechanism to stop the scheduler.

The main loop of the scheduler is in the `run` method, which is executed in a separate thread.

To communicate to the scheduler that it needs to stop, `threading.Event` object is used.
When `stop` method is called, the internal flag of the `self.stop_event` object is set to `True`.
In the `run` method, *while loop* checks if `stop_event` is set and if yes, exits the loop.

`startup` and `shutdown` methods are run before and after the *while loop*. 
`handle` method contains the business logic of the scheduler.

`SchedulerFactory` provides a convenient method to create a different kinds of schedulers.

Thank you for reading.
