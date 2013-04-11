a test mysql python connector which support tornado ioloop

the package expose two mysql interface:
    `send_query` , `read_query_result`
and connection file descriptor:
    `fd`

so we can import mysql connection to tornado ioloop.

eg:

we extends the default Connection to create a asynchronous EPollConnection

    from tornado import ioloop
    from functools import partial
    from MySQLdb.connections import Connection

    class EPollConnection(Connection):
        """.
            non-blocking mysql connection
            for tornado ioloop
        """
        def epoll_query(self, query, callback, on_error=None, args=None):
            """ Non-blocking query. callback is function that takes list
                of tuple args """
            self.send_query(query)
            ioloop.IOLoop.instance().add_handler(self.fd,
                partial(self._handle_read, callback=callback, on_error=on_error), ioloop.IOLoop.READ)

        def _handle_read(self, fd, ev, callback=None,on_error=None):
            res = []
            try:
                self.read_query_result()
                result = self.use_result()
                while True:
                    row = result.fetch_row()
                    if not row:
                        break
                    res.append(row[0])
                callback(res)
            except Exception, e:
                if on_error:
                    return on_error(e)
                else:
                    raise e
            finally:
                self._cleanup()

        def _cleanup(self):
                ioloop.IOLoop.instance().remove_handler(self.fd)

with EPollConnection we can make async mysql request

    class MainHandler(tornado.web.RequestHandler):
        @tornado.web.asynchronous
        def get(self):
            conn=EPollConnection(host="localhost",user="root",passwd="",db="lomo_production",charset="utf8")
            conn.epoll_query("select sleep(2)",callback=self.do_res)

        def do_res(self,res):
            self.finish('xxx'+json.dumps(res))

the is a test repo, never test in production.
