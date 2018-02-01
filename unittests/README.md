## Test only tested function. Don't test other function via tested function.

```javascript
// my-module.js

var uuid = require("uuid/v4");

exports.download = (src, opts) => {
    opts = opts || {};
    var dst = opts.dst || "/tmp/" + uuid();
    var attempts = opts.attempts || 10;
    exports._download(src, dst, attempts);
    return dst;
};

exports._download = (src, dst, attempts) => {
    // some code to download file
};
```

👎 **poor:**

```javascript
// tests.js

var fs = require("fs");

var expect = require("chai").expect;

var m = require("./my-module");

describe("module", () => {

    describe(".download()", () => {
        it("should download file with default options", () => {
            var destPath = m.download("http://example.com/my-file.jpg");
            expect(fs.existsSync(destPath)).to.be.true;
        });

        it("should download file with custom options", () => {
            var dst = "/path/to/my-file.jpg";
            var destPath = m.download("http://example.com/my-file.jpg",
                                      { dst: dst, attempts: 1 });
            expect(destPath).to.be.equal(dst);
            expect(fs.existsSync(destPath)).to.be.true;
        });
    });
});
```

👍 **good:**

```javascript
// tests.js

var fs = require("fs");

var chai = require("chai");
var expect = chai.expect;
var sinon = require("sinon");
var sinonChai = require("sinon-chai");

chai.use(sinonChai);

var m = require("./my-module");

describe("module", () => {
    var fileUrl = "https://example.com/file.png";
    var sandbox = sinon.createSandbox();

    afterEach(() => {
        sandbox.restore();
    });

    describe(".download()", () => {

        beforeEach(() => {
            sandbox.stub(m, "_download");
        });

        it("should provide default options", () => {
            var destPath = m.download(fileUrl);
            expect(destPath.startsWith("/tmp/")).to.be.true;
            expect(m._download).to.be.calledOnce;
            expect(m._download.args[0][0]).to.be.equal(fileUrl);
            expect(m._download.args[0][1]).to.be.equal(destPath);
            expect(m._download.args[0][2]).to.be.equal(10);
        });

        it("should provide custom options", () => {
            var dst = "/path/to/my-file.png";
            var destPath = m.download(fileUrl, { dst: dst, attempts: 1 });
            expect(m._download).to.be.calledOnce;
            expect(m._download.args[0][0]).to.be.equal(fileUrl);
            expect(m._download.args[0][1]).to.be.equal(dst);
            expect(m._download.args[0][2]).to.be.equal(1);
        });
    });

    describe("._download()", () => {

        it("should download file", () => {
            var dst = "/path/to/my-file.png";
            m._download(fileUrl, dst, 1);
            expect(fs.existsSync(dst)).to.be.true;
        });
    });
});
```

## Use chaijs expect for assertions. Don't use chaijs should.

- expect may pass custom error message
- expect may work with null objects

👎 **poor:**

```javascript
var chai = require("chai");
chai.should();

describe("my tests", () => {

    it("should check number", () => {
        var one = 1;
        one.should.be.equal(1);
    });
});
```

👍 **good:**

```javascript
var expect = require("chai").expect;

describe("my tests", () => {

    it("should check number", () => {
        expect(1).to.be.equal(1);
    });

    it("should check null", () => {
        var path = null;
        expect(path, "Invalid path value").to.exist;
    });
});
```

## Use sinonjs spies, mocks and stubs for time-consuming and system-changing functions. Use them as much as possible.

```javascript
// my-module.js

exports.downloadFile = (src, dst) => {
    exports._checkArgs(src, dst);
    exports._download(src, dst);
};

exports.calculateStars = () => {
    exports._setupTelescope();
    return exports._calculateStars();
};
```

👎 **poor:**

```javascript
// tests.js

var fs = require("fs");
var expect = require("chai").expect;
var m = require("./my-module");

describe("module", () => {

    describe(".downloadFile()", () => {
        it("should download file", () => {
            m.downloadFile("http://example.com/my-file.png",
                           "/path/to/my-file.png");
            expect(fs.existsSync("/path/to/my-file.png")).to.be.true;
        });
    });

    describe(".calculateStars()", () => {
        it("should calculate stars in universe", () => {
            expect(m.calculateStars()).to.be.equal(100000000000000000000000000);
        });
    });
});
```

👍 **good:**

```javascript
// tests.js

var chai = require("chai");
var expect = chai.expect;
var sinon = require("sinon");
var sinonChai = require("sinon-chai");

sinon.use(sinonChai);

var m = require("./my-module");

describe("module", () => {
    var sandbox = sinon.createSandbox();

    afterEach(() => {
        sandbox.restore();
    });

    describe(".downloadFile()", () => {

        beforeEach(() => {
            sandbox.stub(m, "_checkArgs");
            sandbox.stub(m, "_download");
        });

        it("should download file", () => {
            m.downloadFile("http://example.com/my-file.png",
                           "/path/to/my-file.png");
            expect(m._checkArgs).to.be.calledOnce;
            expect(m._download).to.be.calledOnce;
        });
    });

    describe(".calculateStars()", () => {

        beforeEach(() => {
            sandbox.stub(m, "_setupTelescope");
            sandbox.stub(m, "_calculateStarts").returns(1);
        })

        it("should calculate stars in universe", () => {
            expect(m.calculateStars()).to.be.equal(1);
            expect(m._setupTelescope).to.be.calledOnce;
            expect(m._calculateStarts).to.be.calledOnce;
        });
    });
});
```

## Use sinonjs sandbox to manage mocks.

👎 **poor:**

```javascript
var sinon = require("sinon");
var m = require("./my-module");

describe("my tests", () => {

    beforeEach(() => {
        sinon.stub(m, "download");
        sinon.stub(m, "upload");
    });

    afterEach(() => {
        m.download.restore();
        m.upload.restore();
    });
});
```

👍 **good:**

```javascript
var sinon = require("sinon");
var m = require("./my-module");

describe("my tests", () => {
    var sandbox = sinon.createSandbox();

    beforeEach(() => {
        sandbox.stub(m, "download");
        sandbox.stub(m, "upload");
    });

    afterEach(() => {
        sandbox.restore();
    });
});
```

## Use rewire for monkey patching of internal imports.

```javascript
// my-module.js

var uuid = require("uuid/v4");
var requests = require("requests");

exports.download = (src, opts) => {
    opts = opts || {};
    var dst = opts.dst || "/tmp/" + uuid();
    var attempts = opts.attempts || 10;
    requests.download(src, dst, attempts);
    return dst;
};
```

👎 **poor:**

```javascript
// tests.js

var fs = require("fs");
var expect = require("chai").expect;
var m = require("./my-module");

describe("module", () => {

    describe(".download()", () => {
        it("should download file with default options", () => {
            var destPath = m.download("http://example.com/my-file.jpg");
            expect(fs.existsSync(destPath)).to.be.true;
        });
    });
});
```

👍 **good:**

```javascript
// tests.js

var chai = require("chai");
var expect = chai.expect;
var rewire = require("rewire");
var sinon = require("sinon");
var sinonChai = require("sinon-chai");

chai.use(sinonChai);

var m = rewire("./my-module");

describe("module", () => {
    var fileUrl = "https://example.com/file.png";
    var sandbox = sinon.createSandbox();

    afterEach(() => {
        sandbox.restore();
    });

    describe(".download()", () => {
        var requests = m.__get__("requests");

        beforeEach(() => {
            sandbox.stub(requests, "download");
        });

        it("should provide default options", () => {
            var destPath = m.download(fileUrl);
            expect(destPath.startsWith("/tmp/")).to.be.true;
            expect(requests.download).to.be.calledOnce;
            expect(requests.download.args[0][0]).to.be.equal(fileUrl);
            expect(requests.download.args[0][1]).to.be.equal(destPath);
            expect(requests.download.args[0][2]).to.be.equal(10);
        });
    });
});
```

**NOTE!** If in some reasons you can't use rewire, you may use below approaches.

## Use dependency injection for classes.

👎 **poor:**

```javascript
// my-module.js

var requests = require("requests");

var Downloader = function (url) {
    this.url = url;
};

Downloader.prototype.download = function (path) {
    requests.download(this.url, path);
};

module.exports = Downloader;
```

```javascript
// tests.js

var fs = require("fs");
var expect = require("chai").expect;

var Downloader = require("./my-module");

describe("downloader", () => {
    var d;

    beforeEach(() => {
        d = new Downloader("https://example.com/my-file.png");
    });

    describe(".download()", () => {
        it("should download file", () => {
            d.download("/path/to/file.png");
            expect(fs.existsSync("/path/to/file.png")).to.be.true;
        });
    });
});
```

👍 **good:**

```javascript
// my-module.js

var requests = require("requests");

var Downloader = function (url, inject) {
    this.url = url;

    inject = inject || {};
    this.__requests = inject.requests || requests;
};

Downloader.prototype.download = function (path) {
    this.__requests.download(this.url, path);
};

module.exports = Downloader;
```

```javascript
// tests.js

var chai = require("chai");
var expect = chai.expect;
var sinon = require("sinon");
var sinonChai = require("sinon-chai");

var Downloader = require("./my-module");

sinon.use(sinonChai);

describe("downloader", () => {
    var d;

    beforeEach(() => {
        var fakeRequests = { download: sinon.spy() };
        d = new Downloader("https://example.com/my-file.png",
                           { requests: fakeRequests });
    });

    describe(".download()", () => {
        it("should download file", () => {
            d.download("/path/to/file.png");
            expect(d.__requests.download).to.be.calledOnce;
            expect(d.__requests.download.args[0][0]).to.be.equal(d.url);
            expect(d.__requests.download.args[0][1]).to.be.equal("/path/to/file.png");
        });
    });
});
```

## Use module.exports namespace for injection.

👎 **poor:**

```javascript
// my-module.js

var uuid = require("uuid/v4");
var requests = require("requests");

exports.download = (src, opts) => {
    opts = opts || {};
    var dst = opts.dst || "/tmp/" + uuid();
    var attempts = opts.attempts || 10;
    requests.download(src, dst, attempts);
    return dst;
};
```

```javascript
// tests.js

var fs = require("fs");
var expect = require("chai").expect;

var m = require("./my-module");

describe("module", () => {

    describe(".download()", () => {
        it("should download file with default options", () => {
            var destPath = m.download("http://example.com/my-file.jpg");
            expect(fs.existsSync(destPath)).to.be.true;
        });
    });
});
```

👍 **good:**

```javascript
// my-module.js

var self = exports;

var uuid = require("uuid/v4");
self.__requests = require("requests");

self.download = (src, opts) => {
    opts = opts || {};
    var dst = opts.dst || "/tmp/" + uuid();
    var attempts = opts.attempts || 10;
    self.__requests.download(src, dst, attempts);
    return dst;
};
```

```javascript
// tests.js

var chai = require("chai");
var expect = chai.expect;
var sinon = require("sinon");
var sinonChai = require("sinon-chai");

chai.use(sinonChai);

var m = require("./my-module");

describe("module", () => {
    var fileUrl = "https://example.com/file.png";
    var sandbox = sinon.createSandbox();

    afterEach(() => {
        sandbox.restore();
    });

    describe(".download()", () => {

        beforeEach(() => {
            sandbox.stub(m.__requests, "download");
        });

        it("should provide default options", () => {
            var destPath = m.download(fileUrl);
            expect(destPath.startsWith("/tmp/")).to.be.true;
            expect(m.__requests.download).to.be.calledOnce;
            expect(m.__requests.download.args[0][0]).to.be.equal(fileUrl);
            expect(m.__requests.download.args[0][1]).to.be.equal(destPath);
            expect(m.__requests.download.args[0][2]).to.be.equal(10);
        });
    });
});
```

## Move unnamed callbacks to named functions and test them separately. Don't create callback hell.

👎 **poor:**

```javascript
var fs = require("fs");
var https = require("https");

exports.download = (fileUrl, filePath) => {
    var file = fs.createWriteStream(filePath);

    return new Promise((resolve, reject) => {

        https.get(fileUrl, response => {

            response.pipe(file);
            file.on("finish", () => {
                file.close(resolve);
            });

        }).on("error", err => {

            if (fs.existsSync(filePath)) fs.unlinkSync(filePath);
            reject(err);
        });
    });
};
```

👍 **good:**

```javascript
var self = exports;

self.__fs = require("fs");
self.__https = require("https");

self.download = (fileUrl, filePath) => {
    return new Promise(self.__download(fileUrl, filePath));
};

self.__download = (fileUrl, filePath) => (resolve, reject) => {
    self.__https
        .get(fileUrl, self.__getCb(filePath, resolve))
        .on("error", self.__errCb(filePath, reject));
};

self.__getCb = (filePath, resolve) => response => {
    var file = self.__fs.createWriteStream(filePath);
    response.pipe(file);
    file.on("finish", () => file.close(resolve));
};

self.__errCb = (filePath, reject) => err => {
    if (self.__fs.existsSync(filePath)) {
        self.__fs.unlinkSync(filePath);
    };
    reject(err);
};
```

## Move decomposed code to separated module if it contains a plenty of functions.

👎 **poor:**

```javascript
// utils.js

var self = exports;

self.__fs = require("fs");
self.__https = require("https");

self.upload = (filePath, fileUrl) => {
    // upload
};

self.checkOpts = opts => {
    // check
};

self.download = (fileUrl, filePath) => {
    return new Promise(self.__download(fileUrl, filePath));
};

self.__download = (fileUrl, filePath) => (resolve, reject) => {
    self.__https
        .get(fileUrl, self.__getCb(filePath, resolve))
        .on("error", self.__errCb(filePath, reject));
};

self.__getCb = (filePath, resolve) => response => {
    var file = self.__fs.createWriteStream(filePath);
    response.pipe(file);
    file.on("finish", () => file.close(resolve));
};

self.__errCb = (filePath, reject) => err => {
    if (self.__fs.existsSync(filePath)) {
        self.__fs.unlinkSync(filePath);
    };
    reject(err);
};
```

👍 **good:**

```javascript
// utils/index.js

exports.download = require("./download").download;

exports.upload = (filePath, fileUrl) => {
    // upload
};

exports.checkOpts = opts => {
    // check
};
```

```javascript
// utils/download.js

var self = exports;

self.__fs = require("fs");
self.__https = require("https");

self.download = (fileUrl, filePath) => {
    return new Promise(self.__download(fileUrl, filePath));
};

self.__download = (fileUrl, filePath) => (resolve, reject) => {
    self.__https
        .get(fileUrl, self.__getCb(filePath, resolve))
        .on("error", self.__errCb(filePath, reject));
};

self.__getCb = (filePath, resolve) => response => {
    var file = self.__fs.createWriteStream(filePath);
    response.pipe(file);
    file.on("finish", () => file.close(resolve));
};

self.__errCb = (filePath, reject) => err => {
    if (self.__fs.existsSync(filePath)) {
        self.__fs.unlinkSync(filePath);
    };
    reject(err);
};
```

## Move complex logic operations to separated functions.

👎 **poor:**

```javascript
var SshClient = function (ssh) {
    this.ssh = ssh;
};

SshClient.prototype.switchConnection = function () {

    if (this.ssh._sock &&
            this.ssh._sock.writable &&
            this.ssh._sshstream &&
            this.ssh._sshstream.writable) {
        this.ssh.close();
    } else {
        this.ssh.connect();
    };
};
```

👍 **good:**

```javascript
var SshClient = function (ssh) {
    this.ssh = ssh;
};

SshClient.prototype.switchConnection = function () {
    if (this._isConnected()) {
        this.ssh.close();
    } else {
        this.ssh.connect();
    };
};

SshClient.prototype._isConnected = function () {
    return this.ssh._sock &&
        this.ssh._sock.writable &&
        this.ssh._sshstream &&
        this.ssh._sshstream.writable
};
```
