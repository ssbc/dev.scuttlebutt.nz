<code-explorer language='javascript'>
const Stack = require('secret-stack')
const caps = require('ssb-caps')
const ssbKeys = require('ssb-keys')
const path = require('path')
const CRUT = require('ssb-crut')
// CRUT = create/ read/ update/ tombstone.
// this module takes simple definitions and builds database methods which work in scuttlebutt's p2p context

const podcastModel = require('./model/podcast')
// definitions which ssb-crut use to build "podcast" methods
const commentModel = require('./model/comment')
// definitions which ssb-crut use to build "comment" methods

module.exports = function ssb (opts = {}) {
  if (!opts.path) opts.path = path.join(__dirname, 'data')
  // where our keys + database will be stored
  if (!opts.keys) {
    const keyPath = path.join(opts.path, 'secret')
    opts.keys = ssbKeys.loadOrCreateSync(keyPath)
    // if feed keys are not provided, we create some and store them in opts.path/secret
    // these are the keys used to identify for feed, sign messages you publish, and for encrypting private messages
  }

  const stack = Stack({ caps })
    // start a new secret-stack instance with default "caps" (capabilities)
    // these are unique network keys - only peers with the same caps can communoicate with each other
    .use(require('ssb-db2'))
    // installs a "database", providing methods like "publish", and "get"
    // this also takes care of validating new messages which we receive from remote peers
    .use(require('ssb-db2/compat/db'))
    .use(require('ssb-db2/compat/history-stream'))
    // makes sure legacy replication method "createHistoryStream" is present
    .use(require('ssb-db2/compat/feedstate'))
    // adds methods needs for ssb-crut to work

  const ssb = stack(opts)

  ssb.podcast = new CRUT(ssb, podcastModel)
  // new calls upon crut to create a new thing, in this case a podcastModel
  // these methods will be ssb.podcast.create etc.
  ssb.comment = new CRUT(ssb, commentModel) //

  return ssb
}
</code-explorer>
